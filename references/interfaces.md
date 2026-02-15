# Interface Design Patterns

## Accept Interfaces, Return Structs

The single most important Go interface rule.

```go
// GOOD — function accepts interface
func ProcessItems(store Store, items []Item) error { ... }

// GOOD — constructor returns concrete type
func NewKeyringStore(ring keyring.Keyring) *KeyringStore { ... }

// GOOD — factory returns interface (multiple implementations)
func OpenDefault() (Store, error) { ... }
```

**Why return concrete types?**
- Callers can access all methods, not just interface subset
- Enables adding methods without breaking interface compatibility
- Avoids unnecessary abstraction

**Why accept interfaces?**
- Enables testing with mocks
- Decouples from specific implementations
- Functions only depend on what they actually use

## Define Interfaces Where They're Used

```go
// WRONG — interface defined next to implementation
package secrets
type Store interface { Get(key string) (string, error) }
type KeyringStore struct { ... }

// RIGHT — interface defined by consumer
package auth
type SecretStore interface { Get(key string) (string, error) }
func NewTokenCache(store SecretStore) *TokenCache { ... }
```

Go interfaces are satisfied implicitly — `KeyringStore` implements `SecretStore`
without knowing about it. This means consumers define exactly what they need.

**Exception:** Shared interfaces used by multiple consumers (like `io.Reader`) are
fine to define centrally.

## Compile-Time Checks

```go
var _ AdminService = (*AdminClient)(nil)
```

This fails at compile time if `AdminClient` doesn't implement `AdminService`.
Place at package level, near the interface definition.

## When NOT to Use Interfaces

- **One implementation, no tests need mocking** — just use the concrete type
- **Data types** — structs with fields, no behavior
- **"Future flexibility"** — YAGNI. Add the interface when you need it
- **Thin wrappers** — if the interface has one method and one implementation, it's ceremony

## Dependency Injection Without Frameworks

### Constructor Injection (Preferred)

```go
type Server struct {
    store  Store      // interface
    logger *slog.Logger
}

func NewServer(store Store, logger *slog.Logger) *Server {
    return &Server{store: store, logger: logger}
}
```

### Provider Pattern (Lazy Init)

```go
type ServiceProvider struct {
    cfg       *config.Config
    adminOnce sync.Once
    admin     AdminService
    adminErr  error
}

func (sp *ServiceProvider) Admin() (AdminService, error) {
    sp.adminOnce.Do(func() {
        sp.admin, sp.adminErr = newAdminClient(sp.cfg)
    })
    return sp.admin, sp.adminErr
}
```

Use when initialization is expensive (HTTP calls, keyring access) and not always needed.

### Package-Level Vars (Test Seams)

```go
var openStore = secrets.OpenDefault  // Production value

// In tests:
openStore = func() (Store, error) { return &mockStore{}, nil }
```

Simpler than interfaces for leaf dependencies. Use `t.Cleanup()` to restore.
