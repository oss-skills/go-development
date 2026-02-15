# Resilience Patterns

## Retry Transport

Wrap `http.RoundTripper` for transparent retries:

```go
type RetryTransport struct {
    Base          http.RoundTripper
    MaxRetries429 int           // Rate limit retries (default: 3)
    MaxRetries5xx int           // Server error retries (default: 1)
    BaseDelay     time.Duration // For exponential backoff (default: 1s)
}

func (t *RetryTransport) RoundTrip(req *http.Request) (*http.Response, error) {
    ensureReplayableBody(req) // Must be able to re-send POST/PUT body

    retries429, retries5xx := 0, 0
    for {
        if req.GetBody != nil {
            req.Body, _ = req.GetBody() // Reset body for retry
        }

        resp, err := t.Base.RoundTrip(req)
        if err != nil {
            return nil, err
        }

        switch {
        case resp.StatusCode < 400:
            return resp, nil // Success

        case resp.StatusCode == 429:
            if retries429 >= t.MaxRetries429 {
                return resp, nil
            }
            drainAndClose(resp.Body)
            sleepCtx(req.Context(), t.backoff(retries429, resp))
            retries429++

        case resp.StatusCode >= 500:
            if retries5xx >= t.MaxRetries5xx {
                return resp, nil
            }
            drainAndClose(resp.Body)
            sleepCtx(req.Context(), t.BaseDelay)
            retries5xx++

        default:
            return resp, nil // 4xx (except 429): don't retry
        }
    }
}
```

## Replayable Body

POST/PUT requests need body replay for retries:

```go
func ensureReplayableBody(req *http.Request) {
    if req.Body == nil || req.GetBody != nil {
        return
    }
    bodyBytes, _ := io.ReadAll(req.Body)
    req.Body.Close()

    req.GetBody = func() (io.ReadCloser, error) {
        return io.NopCloser(bytes.NewReader(bodyBytes)), nil
    }
    req.Body, _ = req.GetBody()
}
```

## Exponential Backoff with Jitter

```go
func (t *RetryTransport) backoff(attempt int, resp *http.Response) time.Duration {
    // 1. Respect Retry-After header
    if ra := resp.Header.Get("Retry-After"); ra != "" {
        if seconds, err := strconv.Atoi(ra); err == nil {
            return time.Duration(seconds) * time.Second
        }
        if t, err := http.ParseTime(ra); err == nil {
            return time.Until(t)
        }
    }

    // 2. Exponential backoff: 1s, 2s, 4s, 8s...
    base := t.BaseDelay * time.Duration(1<<attempt)

    // 3. Add jitter (0 to 50% of base) to prevent thundering herd
    jitter := time.Duration(rand.Int64N(int64(base / 2)))

    return base + jitter
}
```

**Why jitter matters:** Without jitter, all clients retry at the same time after
a rate limit, causing another spike. Random jitter spreads retries over time.

## Circuit Breaker

Prevents hammering a failing service:

```go
type CircuitBreaker struct {
    mu          sync.Mutex
    failures    int
    lastFailure time.Time
    open        bool
    threshold   int           // Open after N failures
    resetTime   time.Duration // Try again after this duration
}

func (cb *CircuitBreaker) RecordSuccess() {
    cb.mu.Lock()
    defer cb.mu.Unlock()
    cb.failures = 0
    cb.open = false
}

func (cb *CircuitBreaker) RecordFailure() {
    cb.mu.Lock()
    defer cb.mu.Unlock()
    cb.failures++
    cb.lastFailure = time.Now()
    if cb.failures >= cb.threshold {
        cb.open = true
    }
}

func (cb *CircuitBreaker) IsOpen() bool {
    cb.mu.Lock()
    defer cb.mu.Unlock()
    if !cb.open {
        return false
    }
    // Auto-reset after timeout (half-open state)
    if time.Since(cb.lastFailure) > cb.resetTime {
        cb.open = false
        cb.failures = 0
        return false
    }
    return true
}
```

**States:** Closed (normal) → Open (failing, reject requests) → Half-Open (try one request after reset timeout)

## Drain and Close

Always drain response bodies before closing, even on error:

```go
func drainAndClose(body io.ReadCloser) {
    if body == nil { return }
    io.Copy(io.Discard, body) // Drain to allow connection reuse
    body.Close()
}
```

HTTP/1.1 connections can't be reused if the body isn't fully read.

## Composing Transports

Layer behaviors with the `http.RoundTripper` interface:

```go
baseTransport := http.DefaultTransport
oauthTransport := &oauth2.Transport{Source: tokenSource, Base: baseTransport}
retryTransport := &RetryTransport{Base: oauthTransport, MaxRetries429: 3}

client := &http.Client{Transport: retryTransport}
```

Each layer wraps the next: retry → oauth → base HTTP.
