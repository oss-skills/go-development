---
name: go-development
description: >
  Production Go development patterns: error handling (typed errors, sentinel errors,
  Unwrap chains), interface design (accept interfaces, return structs), testing (table-driven,
  test seams, httptest), concurrency (bounded parallelism, timeouts), resilience (circuit
  breaker, exponential backoff), and project tooling (golangci-lint, gofumpt, Makefile).
  Use when writing, reviewing, or setting up Go code and projects. Based on patterns from
  gogcli and Effective Go.
---

# Go Development

Production Go patterns extracted from real codebases. Not generic advice — concrete
patterns with rationale.

## When to Use

- Writing or reviewing Go code
- Designing interfaces, error types, or package structure
- Setting up Go project tooling (linting, formatting, CI)
- Implementing concurrent or resilient HTTP clients
- Writing tests for Go code

## When NOT to Use

- CLI-specific patterns (Kong, output formatting, exit codes) — use `open-cli` skill
- Trivial scripts or one-off tools

## Project Structure

```
cmd/<binary>/main.go       # Thin: just calls Execute() and handles exit
internal/                  # All implementation — prevents external imports
  <domain>/                # Package per concern, not per layer
go.mod
Makefile
```

**`main.go` should be ~10 lines:**
```go
func main() {
    if err := cmd.Execute(os.Args[1:]); err != nil {
        os.Exit(exitCode(err))
    }
}
```

**`internal/` enforces encapsulation** — consumers can't import your implementation.
Package by domain (`auth/`, `config/`, `zoho/`), not by layer (`models/`, `services/`).

## Error Handling

See [errors.md](./references/errors.md) for full patterns.

### Typed Errors

```go
type AuthRequiredError struct {
    Service string
    Email   string
    Cause   error
}

func (e *AuthRequiredError) Error() string {
    return fmt.Sprintf("auth required for %s %s", e.Service, e.Email)
}

func (e *AuthRequiredError) Unwrap() error { return e.Cause }

// Checker function — consumers use this, not type assertions
func IsAuthRequiredError(err error) bool {
    var e *AuthRequiredError
    return errors.As(err, &e)
}
```

### Sentinel Errors

```go
var (
    errNotFound     = errors.New("not found")
    errStateMismatch = errors.New("state mismatch")
)

// Check with errors.Is — works through wrapping chains
if errors.Is(err, errNotFound) { ... }
```

### Wrapping Rules

- **Wrap with context:** `fmt.Errorf("fetch users: %w", err)`
- **Never wrap twice:** don't wrap an already-wrapped error with the same context
- **Actionable messages:** `"no refresh token; try again with --force-consent"`

## Interface Design

See [interfaces.md](./references/interfaces.md) for full patterns.

### Accept Interfaces, Return Structs

```go
// Define interface WHERE IT'S USED, not where implemented
type Store interface {
    Get(key string) (string, error)
    Set(key, value string) error
}

// Constructor returns concrete type (or interface if multiple implementations)
func NewKeyringStore(ring keyring.Keyring) *KeyringStore {
    return &KeyringStore{ring: ring}
}

// Factory returns interface when implementation varies
func OpenDefault() (Store, error) {
    ring, err := openKeyring()
    if err != nil { return nil, err }
    return &KeyringStore{ring: ring}, nil
}
```

### Compile-Time Interface Checks

```go
var _ Store = (*KeyringStore)(nil)
var _ Store = (*FileStore)(nil)
```

### When NOT to Use Interfaces

- Only one implementation exists and you don't need test mocking
- The type is a simple data struct
- You're adding indirection for "future flexibility" — YAGNI

## Testing

See [testing.md](./references/testing.md) for full patterns.

### Table-Driven Tests

```go
func TestFormatBytes(t *testing.T) {
    tests := []struct {
        name     string
        bytes    int64
        expected string
    }{
        {name: "zero", bytes: 0, expected: "0 B"},
        {name: "KB boundary", bytes: 1024, expected: "1.0 KB"},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            assert.Equal(t, tt.expected, formatBytes(tt.bytes))
        })
    }
}
```

### Package-Level Vars as Test Seams

Alternative to interfaces for dependency injection:

```go
// Production code
var openSecretsStore = secrets.OpenDefault

func createClient() (*Client, error) {
    store, err := openSecretsStore()
    // ...
}

// Test code
func TestCreateClient(t *testing.T) {
    orig := openSecretsStore
    t.Cleanup(func() { openSecretsStore = orig })

    openSecretsStore = func() (secrets.Store, error) {
        return &mockStore{}, nil
    }
    // ...
}
```

### Test Helpers

```go
func captureStdout(t *testing.T, fn func()) string {
    t.Helper() // Stack traces point to caller, not here
    r, w, _ := os.Pipe()
    orig := os.Stdout
    os.Stdout = w
    fn()
    w.Close()
    os.Stdout = orig
    b, _ := io.ReadAll(r)
    return string(b)
}
```

## Concurrency

See [concurrency.md](./references/concurrency.md) for full patterns.

### Bounded Parallelism

```go
const maxConcurrency = 10
sem := make(chan struct{}, maxConcurrency)

type result struct {
    index int
    item  Item
    err   error
}
results := make(chan result, len(items))
var wg sync.WaitGroup

for i, item := range items {
    wg.Add(1)
    go func(idx int, it Item) {
        defer wg.Done()
        select {
        case sem <- struct{}{}:
            defer func() { <-sem }()
        case <-ctx.Done():
            results <- result{index: idx, err: ctx.Err()}
            return
        }
        // ... do work
        results <- result{index: idx, item: processed}
    }(i, item)
}

go func() { wg.Wait(); close(results) }()
```

### Timeout Pattern

```go
func openWithTimeout(timeout time.Duration) (Resource, error) {
    ch := make(chan resourceResult, 1)
    go func() {
        r, err := openBlocking()
        ch <- resourceResult{r, err}
    }()
    select {
    case res := <-ch:
        return res.resource, res.err
    case <-time.After(timeout):
        return nil, fmt.Errorf("timeout after %v", timeout)
    }
}
```

## Resilience

See [resilience.md](./references/resilience.md) for full patterns.

### Retry with Exponential Backoff

- Respect `Retry-After` header first
- Exponential backoff with jitter: `baseDelay * 2^attempt + random(0, baseDelay/2)`
- Separate retry limits per error class (3 for 429, 1 for 5xx)
- Make request body replayable for POST/PUT retries
- Context-aware sleep (respect cancellation during backoff)

### Circuit Breaker

- Open after N consecutive failures
- Auto-reset after timeout period
- `sync.Mutex` for thread safety
- Log state transitions

## File I/O

### Atomic Writes

```go
func writeConfig(path string, data []byte) error {
    tmp := path + ".tmp"
    if err := os.WriteFile(tmp, data, 0o600); err != nil {
        return fmt.Errorf("write: %w", err)
    }
    return os.Rename(tmp, path) // Atomic on Unix
}
```

### Permissions

- Config files: `0o600` (user read/write only)
- Config dirs: `0o700` (user access only)
- Never store secrets in config — use keyring or encrypted file store

## Context Propagation

### Context Values with Type-Safe Keys

```go
type ctxKey struct{} // Empty struct — unique per package

func WithMode(ctx context.Context, mode Mode) context.Context {
    return context.WithValue(ctx, ctxKey{}, mode)
}

func FromContext(ctx context.Context) Mode {
    if v, ok := ctx.Value(ctxKey{}).(Mode); ok {
        return v
    }
    return Mode{} // Zero value default
}
```

**Rules:**
- One key type per package (empty struct prevents collisions)
- Helper functions hide `context.Value` complexity
- Always provide a safe default for missing values

## Project Tooling

### Makefile

```makefile
VERSION := $(shell git describe --tags --always --dirty 2>/dev/null || echo dev)
LDFLAGS := -X main.version=$(VERSION)
TOOLS_DIR := $(CURDIR)/.tools

build:
	go build -ldflags "$(LDFLAGS)" -o bin/app ./cmd/app

test:
	go test ./... -race -count=1

lint: tools
	$(TOOLS_DIR)/golangci-lint run

fmt: tools
	$(TOOLS_DIR)/goimports -local github.com/your/module -w .
	$(TOOLS_DIR)/gofumpt -w .

tools:
	@mkdir -p $(TOOLS_DIR)
	GOBIN=$(TOOLS_DIR) go install mvdan.cc/gofumpt@v0.9.2
	GOBIN=$(TOOLS_DIR) go install golang.org/x/tools/cmd/goimports@v0.41.0
	GOBIN=$(TOOLS_DIR) go install github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.8.0

ci: fmt lint test
```

**Key points:**
- **Local tools in `.tools/`** — don't pollute global `GOPATH/bin`
- **Pinned versions** — reproducible builds
- **`-local` for goimports** — groups internal imports separately
- **`-race` in tests** — always detect data races
- **Version via ldflags** — embed git info at build time

## References

| File | Topic |
|------|-------|
| [errors.md](./references/errors.md) | Typed errors, sentinel errors, wrapping, error checking |
| [interfaces.md](./references/interfaces.md) | Interface design, DI patterns, when to use interfaces |
| [testing.md](./references/testing.md) | Table-driven tests, test seams, httptest, helpers |
| [concurrency.md](./references/concurrency.md) | Bounded parallelism, timeouts, channels, sync primitives |
| [resilience.md](./references/resilience.md) | Circuit breaker, retry transport, backoff, replayable body |

---
*Patterns extracted from [gogcli](https://github.com/steipete/gogcli) by Peter Steinberger and Go standard library idioms.*
