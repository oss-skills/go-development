# Testing Patterns

## Table-Driven Tests

The standard Go pattern. Every test with 3+ cases should be table-driven.

```go
func TestTruncate(t *testing.T) {
    tests := []struct {
        name   string
        input  string
        max    int
        expect string
    }{
        {name: "shorter", input: "hi", max: 10, expect: "hi"},
        {name: "exact", input: "hello", max: 5, expect: "hello"},
        {name: "truncated", input: "hello world", max: 8, expect: "hello..."},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Truncate(tt.input, tt.max)
            assert.Equal(t, tt.expect, got)
        })
    }
}
```

**Rules:**
- Name field is required — shows in test output
- Each case tests ONE behavior, named for what it verifies
- Don't add trivial cases (testing `if true { "Yes" }` is waste)

## Test Helpers

Always mark helpers with `t.Helper()`:

```go
func createTestServer(t *testing.T, handler http.Handler) *httptest.Server {
    t.Helper()
    srv := httptest.NewServer(handler)
    t.Cleanup(srv.Close)
    return srv
}
```

`t.Helper()` makes stack traces point to the calling test, not the helper.
`t.Cleanup()` handles teardown automatically.

## Package-Level Vars as Test Seams

When full interfaces are overkill:

```go
// production.go
var readConfig = config.ReadFromDisk

func Initialize() error {
    cfg, err := readConfig()
    // ...
}

// production_test.go
func TestInitialize(t *testing.T) {
    orig := readConfig
    t.Cleanup(func() { readConfig = orig })

    readConfig = func() (Config, error) {
        return Config{Region: "us"}, nil
    }

    err := Initialize()
    assert.NoError(t, err)
}
```

**When to use this vs interfaces:**
- Test seams: leaf dependencies, simple functions, internal code
- Interfaces: cross-package boundaries, multiple methods, public API

## httptest for API Clients

```go
func TestListUsers(t *testing.T) {
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        assert.Equal(t, "/api/users", r.URL.Path)
        assert.Equal(t, "GET", r.Method)

        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(map[string]any{
            "status": map[string]any{"code": 200},
            "data":   []map[string]any{{"id": 1, "name": "Alice"}},
        })
    }))
    t.Cleanup(srv.Close)

    client := NewClient(srv.URL)
    users, err := client.ListUsers(context.Background())
    assert.NoError(t, err)
    assert.Len(t, users, 1)
}
```

### Composable Test Middleware

```go
func withAuth(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.Header.Get("Authorization") == "" {
            http.Error(w, "unauthorized", 401)
            return
        }
        next.ServeHTTP(w, r)
    })
}

// Usage
srv := httptest.NewServer(withAuth(myHandler))
```

## I/O Capture

For testing functions that write to stdout/stderr:

```go
func captureStdout(t *testing.T, fn func()) string {
    t.Helper()
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

## testify vs stdlib

Use `testify/assert` for cleaner assertions:

```go
// stdlib — verbose, unclear on failure
if got != want {
    t.Errorf("got %v, want %v", got, want)
}

// testify — concise, better error messages
assert.Equal(t, want, got)
assert.NoError(t, err)
assert.Contains(t, output, "expected substring")
```

Use `require` (not `assert`) when failure should stop the test:
```go
require.NoError(t, err) // stops test immediately on failure
```
