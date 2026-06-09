# Chapter 35: Effective Go & Style

## 1. Basic Usage

Idiomatic Go reads like a well‑crafted CLI tool: explicit, predictable, and minimally surprising. The mechanics are simple, but their assembly defines the "Go way."

```go
package user

import (
    "context"
    "errors"
    "fmt"
    "strings"
    "time"
)

// User represents a registered user.
// Field order reflects memory layout (hot fields first).
type User struct {
    ID        int64     // Exported: accessible to other packages
    CreatedAt time.Time
    Email     string
    Name      string
    isActive  bool      // unexported: internal detail
}

// NewUser is a constructor returning a concrete type (not an interface).
// Prefer "New" prefix for constructors.
func NewUser(email, name string) (*User, error) {
    if email = strings.TrimSpace(email); email == "" {
        return nil, errors.New("user: email cannot be empty")
    }
    if name = strings.TrimSpace(name); name == "" {
        return nil, errors.New("user: name cannot be empty")
    }
    return &User{
        ID:        time.Now().UnixNano(), // simplistic, avoid in production
        CreatedAt: time.Now().UTC(),
        Email:     email,
        Name:      name,
        isActive:  true,
    }, nil
}

// Activate sets the user as active.
// Value receiver is fine because we don't modify the original.
// But to modify field, we use pointer receiver.
func (u *User) Activate() {
    u.isActive = true
}

// Deactivate – same pattern.
func (u *User) Deactivate() {
    u.isActive = false
}

// IsActive – use "Is" prefix for boolean getters.
func (u *User) IsActive() bool {
    return u.isActive
}

// String implements fmt.Stringer for controlled representation.
// Never expose internal fields blindly.
func (u User) String() string {
    return fmt.Sprintf("User(id=%d, email=%s, active=%t)", u.ID, u.Email, u.isActive)
}

// Validate performs business rule checks.
// Returning a slice of errors is idiomatic for multi-field validation.
func (u *User) Validate() []error {
    var errs []error
    if !strings.Contains(u.Email, "@") {
        errs = append(errs, errors.New("email missing '@'"))
    }
    if len(u.Name) < 2 {
        errs = append(errs, errors.New("name too short"))
    }
    return errs
}

// Repository interface is defined by the consumer (the package that uses it).
// This is the "accept interfaces, return structs" principle.
type Repository interface {
    Save(ctx context.Context, u *User) error
    FindByID(ctx context.Context, id int64) (*User, error)
}

// Service uses the repository interface.
type Service struct {
    repo Repository
}

func NewService(repo Repository) *Service {
    return &Service{repo: repo}
}

func (s *Service) Register(ctx context.Context, email, name string) (*User, error) {
    u, err := NewUser(email, name)
    if err != nil {
        return nil, fmt.Errorf("registration failed: %w", err)
    }
    if errs := u.Validate(); len(errs) > 0 {
        return nil, fmt.Errorf("validation errors: %v", errs)
    }
    if err := s.repo.Save(ctx, u); err != nil {
        return nil, fmt.Errorf("save failed: %w", err)
    }
    return u, nil
}
```

**Key syntax signals:**
- Unexported vs. exported via `lowercase` / `Uppercase`.
- Constructor always returns a concrete type (not an interface).
- Methods with pointer receivers when mutation is needed.
- `String()` method for logging / debugging (never for serialization).
- Consumer‑defined interfaces – the `Repository` lives in the package that needs it, not in the data layer.

---

## 2. Under the Hood

### Naming and the `internal` Package
The Go compiler enforces visibility at the package level, but also at the filesystem level. Any package inside a directory named `internal/` can only be imported by code inside the parent of that `internal` directory. This is a compile‑time restriction, not a convention.

```
myproject/
├── internal/
│   └── auth/          → only importable by myproject/...
├── cmd/
│   └── server/        → can import myproject/internal/auth
└── pkg/
    └── api/           → can also import myproject/internal/auth
```

This replaces the "`pkg/` is public, `internal/` is private" mental model. The Go team recommends `internal/` for truly private implementation details that you are willing to break without notice.

### Method Sets and Promotion
When you embed a type, the method set of the embedded type is **promoted** to the outer type – but only if the embed is by value or by pointer appropriately.

```go
type Logger struct{}
func (Logger) Log(msg string) { println(msg) }

type Server struct {
    Logger      // value embed → Log promoted with value receiver semantics
    addr string
}

type ServerWithPointer struct {
    *Logger     // pointer embed → Log promoted, but nil receiver possible
}
```

Under the hood, promotion is syntactic sugar. The compiler generates wrapper methods that forward calls. This is not inheritance – it's **delegation without boilerplate**.

### Interface Table (Itable) Layout
When you assign a concrete type to an interface, Go builds an **itable** – a pair of pointers: one to the concrete type’s method table, one to the interface’s method table. Calls through the interface are two indirect jumps: `itable->method->concrete`.

For performance, Go caches itables in a global map keyed by (concrete type, interface type). That’s why the first call to an interface is slightly more expensive – the cache is populated.

### Zero‑Allocation String Conversion
Go’s compiler optimizes `string(b)` where `b` is a `[]byte` if it can prove the byte slice is not mutated afterwards. However, the reverse `[]byte(s)` for a `string` always allocates – unless you use `unsafe` (not recommended for style). For high‑performance path, prefer `strings.Builder` or `bytes.Buffer`.

---

## 3. Why This Design?

### No `implements` Declaration
Go’s implicit interface satisfaction is the single most controversial design choice. The reason: **decoupling**. In Java, if you define an interface `Reader`, every implementation must write `implements Reader`. That creates a source dependency from the implementation to the interface. In Go, the interface is defined by the **consumer** (the function that needs it). The producer never knows about the interface. This enables:

- Mocking without a mocking framework: define a small interface matching the methods you need.
- Testing across package boundaries without circular imports.
- Adding interfaces after the fact without modifying existing code.

### `gofmt` as a Non‑Negotiable Standard
`gofmt` isn't just a tool – it's a **social contract**. The Go team made formatting completely automated so that debates about brace placement, tabs vs. spaces, and indentation size would end forever. The output of `gofmt` is the **canonical source**. Every Go developer, every CI system, every editor plugin uses the same rules. This reduces cognitive load when reading unfamiliar code.

### Why No Default Method Arguments or Function Overloading?
Both features add complexity to the **call site** and to **method resolution**. Go’s philosophy: explicit > implicit. If you need optional parameters, define a `Config` struct and pass it. If you need multiple signatures, use different function names (`MarshalJSON`, `MarshalXML`). This makes code grep‑able and self‑documenting.

### The `error` Interface Deliberately Simple
`error` has one method: `Error() string`. No stack trace, no error codes, no cause chain. That’s a minimal viable interface. Wrapping (`%w`) and unwrapping (`errors.Is/As`) were added later as library features, not language changes. This keeps the core simple while allowing rich error handling as a library concern.

---

## 4. Competing Approaches

| **Language** | **Idiomatic Style** | **How Go Differs** |
|--------------|---------------------|--------------------|
| **Java**     | Deep inheritance trees, `@Override`, checked exceptions, `getter/setter` boilerplate. | Go rejects inheritance, has no `throws`, encourages direct field access. |
| **Python**   | Duck typing at runtime, dynamic attribute creation, heavy reliance on `*args/**kwargs`. | Go’s duck typing is **compile‑time**, no runtime surprises. No variable arguments overloading. |
| **Rust**     | Explicit lifetimes, `impl Trait`, `derive` macros, `cargo fmt` (similar to `gofmt`). | Go has no lifetime annotations – GC eliminates that complexity. Macros are absent; Go prefers code generation (`go:generate`). |
| **C++**      | Operator overloading, multiple inheritance, SFINAE / concepts. | Go has none of these – the type system is deliberately shallow to keep compile times low and readability high. |
| **JavaScript/TS** | Dynamic objects, `.bind(this)`, class sugar, `Prettier` for formatting. | Go has no classes, no `this` rebinding, and formatting is standardised by the toolchain, not a third‑party formatter. |

**Concrete example:** A `Reader` interface in Go is one or two methods. In Java’s standard library, `Reader` is an abstract class with 10+ methods plus `read()` overloads. The Go version is easier to implement correctly.

---

## 5. Common Mistakes

### Mistake 1: Overusing `any` (aka `interface{}`)
```go
// Bad: Type-unsafe, forces reflection or type assertions.
func Process(data any) {
    // ...
}

// Good: Use generics or specific interfaces.
func Process[T fmt.Stringer](data T) {
    fmt.Println(data.String())
}
```
`any` has its place (e.g., `json.Unmarshal`), but as a default for "this could be anything" it defeats static typing.

### Mistake 2: Exporting Implementation Details
```go
// Bad: Client can see and modify internal cache.
type Cache struct {
    store map[string]string  // exported field
    mu    sync.Mutex
}

// Good: Expose only what's needed.
type Cache struct {
    store map[string]string
    mu    sync.Mutex
}

func (c *Cache) Get(key string) (string, bool) { ... }
```

### Mistake 3: Returning Interfaces from Constructors
```go
// Bad: Consumer cannot use concrete methods without type assertion.
type UserRepo interface { ... }
func NewUserRepo() UserRepo { return &postgresRepo{} }

// Good: Return the concrete type.
func NewUserRepo() *postgresRepo { return &postgresRepo{} }
```
Returning an interface locks you into that interface forever. Returning a struct allows adding methods later without breaking callers.

### Mistake 4: Ignoring `context.Context` Propagation
```go
// Bad: No way to cancel or timeout.
func FetchUser(id int64) (*User, error) {
    // ...
}

// Good: Context as first parameter (by convention).
func FetchUser(ctx context.Context, id int64) (*User, error) {
    // ...
}
```

### Mistake 5: Short Variable Names Gone Too Far
While `ctx` is fine, `c` for context is not (too cryptic). Single‑letter variables are allowed only for very short scopes (loop indices, `r` for `http.Request`). The rule: **the larger the scope, the more descriptive the name**.

---

## 6. Performance Considerations

### Interface Call Overhead
A direct method call on a concrete type is a single `CALL` instruction. A call through an interface adds two indirect jumps (itable lookup + function pointer). This is about 5‑10ns on modern CPUs – negligible in I/O‑bound code but measurable in hot loops.

```go
type Fast struct{}
func (Fast) Work() {}

type Worker interface { Work() }

var sink int

func BenchmarkDirect(b *testing.B) {
    f := Fast{}
    for i := 0; i < b.N; i++ {
        f.Work()
    }
}

func BenchmarkInterface(b *testing.B) {
    var w Worker = Fast{}
    for i := 0; i < b.N; i++ {
        w.Work()
    }
}
// Interface is roughly 2–3x slower (but still < 20ns per op).
```

### Inlining and Escape Analysis
The Go compiler inlines small functions (≤ a certain cost). Interface dispatches **cannot be inlined** because the callee is unknown at compile time. Similarly, methods called via a pointer receiver often escape arguments to the heap. Value receivers are preferred for small, immutable types.

### String Concatenation
`+` on strings in a loop is O(n²) because strings are immutable. Use `strings.Builder` which pre‑allocates in exponential steps:

```go
var b strings.Builder
b.Grow(estimatedSize) // optional hint
for _, s := range list {
    b.WriteString(s)
}
result := b.String()
```

### Package Initialization (`init()`)
Multiple `init()` functions in a package run in declaration order. They are convenient but hurt startup latency and testability. Prefer explicit constructors (`New...`). If `init()` is unavoidable, keep it trivial (e.g., setting `math/rand` seed).

---

## 7. Best Practices (The Idiomatic Way)

### 1. Use `go fmt` and `go vet` in CI
Never merge code that hasn’t been formatted. `go vet` catches suspicious constructs (e.g., `printf`‑style format mismatches, unreachable code).

### 2. Name Packages After Their Purpose, Not Their Content
- `http` is better than `network` or `protocols`.
- `json` is better than `serialization`.
Avoid generic names like `util`, `common`, `base`. These become dumping grounds.

### 3. Keep `main()` Minimal
The `main` package should only parse flags, construct dependencies, and start the server. Everything else lives in separate packages.

### 4. Use `errgroup` for Concurrent Error Handling
```go
import "golang.org/x/sync/errgroup"

g, ctx := errgroup.WithContext(ctx)
g.Go(func() error { return fetchUser(ctx, id) })
g.Go(func() error { return fetchPosts(ctx, userID) })
if err := g.Wait(); err != nil {
    return err
}
```

### 5. Prefer `:=` for Declaration and Initialization
But avoid shadowing. Shadowing happens when you reuse a variable name in an inner scope:

```go
x := 1
if true {
    x := 2 // shadows outer x
    fmt.Println(x) // 2
}
fmt.Println(x) // 1 (bug waiting to happen)
```

Enable `go vet -shadow` (though it’s now part of `go vet` by default in recent versions).

### 6. Single Method Interfaces Are Powerful
Name them with `-er` suffix: `Reader`, `Writer`, `Closer`, `Stringer`, `Formatter`. These small interfaces compose easily:

```go
type ReadCloser interface {
    Reader
    Closer
}
```

### 7. Don't Create Interfaces Just for Testing (If a Real Implementation Is Simple)
If your “real” dependency is a `http.Client`, don’t wrap it in an interface. Use the real client in tests with a test server (`httptest.NewServer`). Only abstract when you need to mock non‑deterministic behavior (e.g., time, random, external API with complex state).

### 8. Use `option` Pattern for Complex Constructors
When a struct has many optional parameters, use functional options:

```go
type Server struct {
    addr    string
    timeout time.Duration
    logger  *slog.Logger
}

type Option func(*Server)

func WithTimeout(d time.Duration) Option {
    return func(s *Server) { s.timeout = d }
}

func WithLogger(l *slog.Logger) Option {
    return func(s *Server) { s.logger = l }
}

func NewServer(addr string, opts ...Option) *Server {
    s := &Server{addr: addr, timeout: 30 * time.Second} // default
    for _, opt := range opts {
        opt(s)
    }
    return s
}
```

### 9. Document Exported Names with Complete Sentences
`go doc` renders the first line of the comment as a summary. Start with the name being documented:

```go
// User represents a registered account. It contains ...
type User struct { ... }

// NewUser validates and creates a User. It returns an error if email is invalid.
func NewUser(email, name string) (*User, error) { ... }
```

### 10. Avoid Globals, Except for `sync.Once` Initialization
Global variables break test isolation and hide dependencies. If you must have a global (e.g., a database connection pool), initialise it in `init()` or `main()` and pass it explicitly.

---

## 8. Examples

### Example 1: A Small, Composable Interface Set

```go
// writer.go
package storage

type Writer interface {
    Write(ctx context.Context, key string, data []byte) error
}

type Closer interface {
    Close() error
}

type WriteCloser interface {
    Writer
    Closer
}
```

### Example 2: Consumer‑Defined Interface for Testing

```go
// service.go
package payment

type Processor interface {
    Charge(ctx context.Context, amount int) (string, error)
}

type Service struct {
    proc Processor
}

// service_test.go
package payment

type mockProcessor struct {
    chargeFunc func(ctx context.Context, amount int) (string, error)
}

func (m *mockProcessor) Charge(ctx context.Context, amount int) (string, error) {
    return m.chargeFunc(ctx, amount)
}

func TestService(t *testing.T) {
    proc := &mockProcessor{
        chargeFunc: func(ctx context.Context, amount int) (string, error) {
            return "tx_123", nil
        },
    }
    svc := NewService(proc)
    // test...
}
```

### Example 3: Graceful Shutdown with Context

```go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()

    srv := &http.Server{Addr: ":8080"}
    go func() {
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()

    <-ctx.Done()
    stop()
    log.Println("shutting down gracefully...")
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    if err := srv.Shutdown(shutdownCtx); err != nil {
        log.Fatal(err)
    }
}
```

### Example 4: Functional Options in a Production Package

```go
package redis

type Config struct {
    Addr     string
    Password string
    DB       int
    PoolSize int
}

type Option func(*Config)

func WithPassword(pw string) Option {
    return func(c *Config) { c.Password = pw }
}

func WithDB(db int) Option {
    return func(c *Config) { c.DB = db }
}

func NewClient(opts ...Option) *Client {
    cfg := &Config{
        Addr:     "localhost:6379",
        PoolSize: 10,
    }
    for _, opt := range opts {
        opt(cfg)
    }
    // ...
}
```

---

## 9. Summary & Exercises

### Summary
- **Idiomatic Go** prioritises readability over cleverness. `gofmt` enforces a single style.
- **Exported vs. unexported** is your only access control – use it deliberately.
- **Consumer‑defined interfaces** reduce coupling and enable testing without heavy mocks.
- **Return concrete types, accept interfaces** – a mantra that prevents abstraction leaks.
- **Small interfaces** (one or two methods) are more powerful than large ones because they compose.
- **Functional options** and `context.Context` are the standard patterns for extensible constructors and cancellation.
- Avoid `any`, `init()` (for complex logic), and returning interfaces from constructors.

### Exercises

#### Exercise 1: Refactor a Java‑style Hierarchy into Go
Given the following (pseudo‑Java) hierarchy:
```java
abstract class Animal { abstract void speak(); }
class Dog extends Animal { void speak() { bark(); } }
class Cat extends Animal { void speak() { meow(); } }
```
Refactor into Go using interfaces and composition. Then add a `Zoo` type that can accept any `Animal` and call `speak()`. Ensure your design allows adding `Bird` without changing `Zoo`.

#### Exercise 2: Build a Thread‑Safe Cache with Expiration
Implement a cache that:
- Stores `string` keys and `any` values.
- Supports `Get`, `Set`, and `Delete`.
- Each entry has a **time‑to‑live (TTL)**; expired entries are removed lazily (on access) and periodically by a background goroutine.
- Use a `sync.RWMutex` for fine‑grained locking.
- Write a benchmark that measures `Get` concurrency (`go test -bench=. -race`).

*Constraints:* Do not use external libraries. Implement the cleanup goroutine with `time.Ticker` and ensure it stops when the cache is no longer used (use `context.Context`).

#### Exercise 3: Design a Package for Retry Logic
Create a package `retry` that exports:
- A `Do` function: `func Do(ctx context.Context, maxAttempts int, fn func() error) error`
- Implements exponential backoff (e.g., 100ms, 200ms, 400ms…).
- Respects context cancellation between attempts.
- Logs each retry attempt with `slog` (structured logging), but make the logger configurable via a functional option.

Write tests that verify:
- A function that always fails retries exactly `maxAttempts` times.
- Context cancellation stops retries early.
- Successful execution on the third attempt returns `nil` and doesn’t retry further.

---

**Aha! Moment:** The “Effective Go” style is not about personal taste – it is the **minimum viable consistency** that makes Go codebases readable by any Go developer, anywhere. By accepting `gofmt`, small interfaces, and explicit error handling, you gain the ability to understand any Go project in minutes, not hours. The style is the **standard library’s discipline** applied to your own code.
