# Error Handling Patterns

## Typed Errors

Use when errors carry domain-specific context that callers need to inspect.

```go
type RateLimitError struct {
    RetryAfter time.Duration
    Attempt    int
}

func (e *RateLimitError) Error() string {
    return fmt.Sprintf("rate limited (retry after %v, attempt %d)", e.RetryAfter, e.Attempt)
}

func (e *RateLimitError) Unwrap() error { return nil }
```

**Always implement `Unwrap()`** — enables `errors.Is/As` to traverse error chains.

**Provide checker functions:**
```go
func IsRateLimitError(err error) bool {
    var e *RateLimitError
    return errors.As(err, &e)
}
```

Callers use `IsRateLimitError(err)` instead of type assertions. This works through
wrapped error chains.

## Sentinel Errors

Use for well-known error conditions without dynamic context:

```go
var (
    errNotFound    = errors.New("resource not found")
    errKeyNotFound = errors.New("key not found in store")
)
```

**Lowercase, unexported** — package-internal. Export only if external packages need
to check for them.

Check with `errors.Is`:
```go
if errors.Is(err, errNotFound) {
    // handle not found
}
```

## Wrapping

```go
// GOOD — adds context about what operation failed
return fmt.Errorf("fetch user %d: %w", id, err)

// BAD — loses the original error type
return fmt.Errorf("error: %v", err)  // %v doesn't wrap

// BAD — redundant wrapping
return fmt.Errorf("failed to fetch: %w", fmt.Errorf("user %d: %w", id, err))
```

**Rules:**
- Use `%w` (not `%v`) to preserve error chains
- Add operation context: what were you trying to do?
- Don't wrap errors from your own package — they already have context
- Include identifiers (user ID, email) when helpful for debugging

## Error → Exit Code Mapping

Map typed errors to stable exit codes in one place:

```go
func exitCode(err error) int {
    if err == nil { return 0 }
    if IsAuthRequiredError(err) { return 4 }
    if IsRateLimitError(err)    { return 7 }
    var notFound *NotFoundError
    if errors.As(err, &notFound) { return 5 }
    return 1 // general error
}
```

| HTTP Status | Exit Code | Meaning |
|-------------|-----------|---------|
| 401 | 4 | Auth required |
| 403 | 6 | Permission denied |
| 404 | 5 | Not found |
| 429 | 7 | Rate limited |
| 5xx | 8 | Server error (retryable) |
