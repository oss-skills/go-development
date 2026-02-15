# Concurrency Patterns

## Bounded Parallelism

Process N items concurrently with a limit of M goroutines:

```go
func fetchAll(ctx context.Context, ids []string) ([]Item, error) {
    const maxConcurrency = 10
    sem := make(chan struct{}, maxConcurrency)

    type result struct {
        index int
        item  Item
        err   error
    }
    results := make(chan result, len(ids))
    var wg sync.WaitGroup

    for i, id := range ids {
        wg.Add(1)
        go func(idx int, itemID string) {
            defer wg.Done()

            // Acquire semaphore (or bail on context cancel)
            select {
            case sem <- struct{}{}:
                defer func() { <-sem }()
            case <-ctx.Done():
                results <- result{index: idx, err: ctx.Err()}
                return
            }

            item, err := fetchOne(ctx, itemID)
            results <- result{index: idx, item: item, err: err}
        }(i, id)
    }

    go func() { wg.Wait(); close(results) }()

    // Collect in order
    ordered := make([]Item, len(ids))
    var firstErr error
    for r := range results {
        if r.err != nil && firstErr == nil {
            firstErr = r.err
        }
        ordered[r.index] = r.item
    }
    if firstErr != nil {
        return nil, firstErr
    }
    return ordered, nil
}
```

**Key elements:**
- **Semaphore channel** — `make(chan struct{}, N)` limits concurrent goroutines
- **Buffered results** — prevent goroutine blocking
- **Index tracking** — preserves input order despite concurrent execution
- **Context check** — respects cancellation at semaphore acquisition

## Timeout with Channel + Select

For operations that might hang (keyring, D-Bus, external processes):

```go
func openWithTimeout(timeout time.Duration) (Resource, error) {
    type result struct {
        resource Resource
        err      error
    }

    ch := make(chan result, 1)
    go func() {
        r, err := openBlocking() // might hang
        ch <- result{r, err}
    }()

    select {
    case res := <-ch:
        return res.resource, res.err
    case <-time.After(timeout):
        return nil, fmt.Errorf("timeout after %v; consider fallback", timeout)
    }
}
```

**Include actionable guidance in timeout errors** — tell users what to do.

## Context-Aware Sleep

For retry backoff that respects cancellation:

```go
func sleepCtx(ctx context.Context, d time.Duration) error {
    select {
    case <-time.After(d):
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

## sync.Once for Lazy Init

Thread-safe single initialization:

```go
type Provider struct {
    once   sync.Once
    client *Client
    err    error
}

func (p *Provider) Get() (*Client, error) {
    p.once.Do(func() {
        p.client, p.err = newClient()
    })
    return p.client, p.err
}
```

**Caveat:** If `Do` panics, subsequent calls will see it as "done" and return
zero values. Handle panics inside the func if the init can fail catastrophically.

## When to Use What

| Pattern | Use Case |
|---------|----------|
| Semaphore channel | Bounded I/O parallelism (API calls, file ops) |
| WaitGroup | Waiting for multiple goroutines to complete |
| Channel + select | Timeout, cancellation, multiplexing |
| sync.Once | Lazy singleton initialization |
| sync.Mutex | Protecting shared state (counters, caches) |
| errgroup.Group | Bounded parallelism with automatic error propagation |
