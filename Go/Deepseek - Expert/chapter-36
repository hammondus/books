# Chapter 36: Common Anti-Patterns

Go’s simplicity is a double-edged sword. The language provides a small, orthogonal set of primitives, yet engineers arriving from ecosystems steeped in design patterns, deep inheritance hierarchies, and heavy frameworks often bring mental models that clash with Go’s philosophy. This chapter catalogs the most pervasive anti-patterns—the “Java-style Go”, the interface excesses, the overengineering that fights the language rather than flowing with it. Each anti-pattern is dissected: how it manifests, why it hurts, and how to refactor it into idiomatic Go that feels right under your fingertips.

---

## 1. Basic Usage

An anti-pattern in Go is any recurring solution that initially looks plausible but ultimately leads to code that is harder to read, slower, or more fragile than the idiomatic alternative. The canonical “basic usage” of anti-pattern avoidance is recognizing when you’re transplanting a habit that the language deliberately discourages.

Consider a trivial “service” that fetches a user by ID. In a language with classes and constructors, you might build:

```go
// ANTI-PATTERN: Java-style constructor injection
type UserService struct {
    repo   *UserRepository
    logger *log.Logger
}

func NewUserService(repo *UserRepository, logger *log.Logger) *UserService {
    return &UserService{repo: repo, logger: logger}
}

func (s *UserService) GetUser(id int) (*User, error) {
    s.logger.Printf("Fetching user %d", id)
    return s.repo.FindByID(id)
}
```

The idiomatic Go version strips away the ceremonial wrapper and makes the dependency explicit at the call site or uses a small interface:

```go
// Idiomatic: accept the precise interface you need.
type UserFinder interface {
    FindByID(id int) (*User, error)
}

func GetUser(ctx context.Context, finder UserFinder, id int) (*User, error) {
    slog.InfoContext(ctx, "Fetching user", "id", id)
    return finder.FindByID(id)
}
```

This isn’t just cosmetic; it removes indirection, eliminates a needless constructor, and makes the function testable with a simple mock that implements `UserFinder`. The “basic usage” of anti-pattern detection is training your eye to spot the extra layers that don’t carry their weight.

---

## 2. Under the Hood

Anti-patterns are not merely aesthetic offenses—they interact poorly with Go’s compiler, runtime, and memory model. Understanding the machinery exposes why some patterns that work well in Java or Python become liabilities here.

**Interface boxing and heap allocations.** When you define an abstract factory returning interfaces for everything, you force the compiler to box concrete values. An interface value is a two-word structure (type pointer, data pointer). Assigning a concrete `*UserService` to an `interface{}` or a large interface causes a heap allocation if the value does not escape analysis trivially. The anti-pattern of preemptively returning interfaces from constructors destroys the compiler’s ability to inline and often pushes data to the heap, increasing GC pressure. Contrast that with the idiomatic rule: “accept interfaces, return structs.” Returning a concrete struct keeps the data on the stack where possible and permits the compiler to inline method calls.

**Goroutine leaks from “fire and forget.”** Patterns that spawn a goroutine without a clear termination signal—often from a constructor that starts a background worker—create dangling goroutines. Each leaked goroutine holds a stack (minimum 2 KiB, growing as needed) and may pin resources. The runtime’s scheduler can handle millions of goroutines, but leaks silently accumulate and eventually cause memory exhaustion or deadlock at shutdown. The anti-pattern of “start a goroutine in `New()`” hides the lifecycle; idiomatic Go explicitly manages goroutine lifecycles with contexts and `sync.WaitGroup`.

**Mutex copying.** Go’s `sync.Mutex` must not be copied after first use. Anti-patterns that embed a mutex in a struct that is passed by value (perhaps because the developer returned the struct, not a pointer) cause the mutex state to be duplicated, rendering locking useless. The `go vet` tool can catch some of these, but the underlying cause is a mental model that treats locks as harmless fields rather than resources with strict ownership semantics.

**`init()` ordering.** Overuse of `init()` functions—often to register drivers or build registries—creates invisible dependency graphs. The Go spec guarantees that `init()` runs after all variable initializations in a package, but the order across packages is deterministic only by import graph. Anti-patterns that rely on “magic” registration via `init()` make startup behavior hard to reason about and impossible to disable for tests. The runtime must execute all `init()` functions before `main`, increasing binary startup latency and coupling code.

These mechanical repercussions show that anti-patterns are not mere style violations; they are correctness and performance hazards amplified by Go’s specific execution model.

---

## 3. Why This Design?

Go’s creators intentionally removed features that enable many traditional patterns, forcing an alternative design space. The language’s design explains why certain anti-patterns emerge and how the language nudges you toward simpler solutions.

**No classes, no inheritance → composition and small interfaces.** Without classes, you cannot build abstract base classes or use template method patterns. Attempting to simulate inheritance with struct embedding often leads to “God structs” that grow too many methods, violating the single-responsibility principle. Go’s answer is composition through embedding for delegation and implicit interfaces for polymorphism. An anti-pattern like `type MyService struct { *BaseService }` where `BaseService` contains all logic only moves the problem; idiomatic Go decomposes behavior into small, focused functions that accept the exact dependencies they need.

**Implicit interfaces → consumer-side definition.** Because Go interfaces are satisfied implicitly, you never need to declare that a struct “implements” an interface in the producer package. The anti-pattern of pre-defining an interface next to the struct (producer-side) robs consumers of the power to define the contract they actually require. The `io.Reader` and `io.Writer` are defined in the standard library precisely because they are consumer abstractions used everywhere. The design teaches that interfaces belong to the package that uses them, not the one that provides the implementation. Overengineering arises when we ignore this and create `UserServiceInterface` in the same file as `UserService`.

**Error values over exceptions → explicit flow.** Go lacks try/catch. Anti-patterns that try to recover from panics for control flow, or that create complex error hierarchies with `errors.Is` chains that mimic exception types, fight the language. The design encourages handling errors where they occur, wrapping them with context, and relying on the simple `error` interface. An anti-pattern is building a custom `AppError` struct with integer codes and severity levels that require the caller to type-assert—this defeats `errors.Is` and `errors.As` and forces brittle coupling.

**Simplicity as a feature.** Rob Pike’s famous quote, “Less is exponentially more,” captures why Go lacks generics (until 1.18), annotations, or aspect-oriented programming. The anti-pattern of heavy use of code generation or reflection to simulate missing features often introduces more complexity than writing a straightforward, slightly repetitive function. The design is patient: a little duplication is acceptable if it preserves clarity and compile-time safety.

Understanding the “why” shifts anti-pattern recognition from a rule to a philosophy: if you’re writing code that feels like a framework, you’re probably swimming upstream.

---

## 4. Competing Approaches

Placing Go anti-patterns in context with other ecosystems illuminates why patterns that are celebrated elsewhere become liabilities.

**Java/C# → excessive abstraction.** Java developers are taught to program to interfaces, use dependency injection containers, and build layers of indirection. In Go, a “service interface” with a single implementation is overhead. The interface does not aid testing if the struct itself can be used directly or a small mock interface defined in the test package. DI containers like Wire or Dagger exist, but the simplest approach—passing dependencies explicitly—works for most programs. The anti-pattern of a `ServiceFactory` returning an `interface` adds indirection that the compiler cannot optimize, and it obscures the concrete type’s methods.

**Python → dynamic typing habits.** Python developers may use global dictionaries as registries, rely on runtime type inspection, or use decorators for middleware. In Go, compile-time safety is paramount; anti-patterns that use `any` with type switches to simulate dynamic dispatch throw away type checking. A Python pattern of catching all exceptions and logging is replaced by explicit `if err != nil`; swallowing an error by assigning it to `_` is a cardinal sin.

**C++/Rust → obsession with ownership and smart pointers.** C++ programmers might use `shared_ptr` for everything, which in Go translates to using pointers for all struct fields regardless of lifetime. Go’s escape analysis often makes value types more efficient; passing a struct by value can keep it on the stack and reduce GC load. The anti-pattern of returning `*MyStruct` from a constructor even when the struct is small and meant to be a value type creates unnecessary heap allocations. Rust’s ownership model doesn’t exist; Go relies on GC, and trying to manually control memory via `sync.Pool` or `unsafe` prematurely is an anti-pattern unless profiling demands it.

**JavaScript/Node.js → async everything.** Node.js developers might use callbacks or promises. Go’s goroutines are so cheap that spawning one per request is idiomatic, but spawning goroutines for every trivial synchronous operation (e.g., a non-blocking map lookup) adds scheduling overhead and complicates error propagation. The anti-pattern of “goroutine scattering”—starting a goroutine without a clear owner that collects its error—leads to silent failures.

The common thread is that Go rewards directness. When you import a pattern from a language that requires ceremony to manage resources, you’re probably adding ceremony Go doesn’t need.

---

## 5. Common Mistakes

This section enumerates concrete anti-patterns with brief illustrative code.

### 5.1 Java-Style Go: Over-Abstraction

- **Producer-side interfaces:**
  ```go
  // ANTI-PATTERN
  type UserStore interface {
      Save(u *User) error
      Find(id string) (*User, error)
  }
  type PostgresUserStore struct{ ... }
  func NewPostgresUserStore() UserStore { ... }
  ```
  The interface is defined in the same package as the implementation. Consumers cannot narrow it; they must import the whole interface. Idiom: define `UserStore` in the consumer package with only the methods it actually uses.

- **Getters and setters for every field:**
  ```go
  func (u *User) GetName() string { return u.name }
  func (u *User) SetName(n string) { u.name = n }
  ```
  In Go, exported fields with zero values are acceptable. Add methods only if you need validation or computed logic.

- **Builder pattern for simple structs:**
  A builder makes sense when construction is complex and has defaults, but `NewUser("alice", "alice@example.com")` is clearer than `UserBuilder{}.Name("alice").Email("...").Build()`. Use functional options for configurable constructors if needed.

### 5.2 Interface Abuse

- **Large interfaces:** `type DoEverything interface { Read(); Write(); Close(); Stat(); ... }` breaks the “small interfaces” rule. The standard library’s `io.Reader` has one method. Break apart monolithic interfaces and let consumers combine them via embedding.

- **Returning an interface from a function “for flexibility”:**
  ```go
  func NewStore() Store { ... }
  ```
  This forces the caller to accept the broad interface even if it needs only a single method. Return the concrete `*PostgresStore` and let callers define their own `Finder` interface.

- **Mocking with a full mock struct in production code:** Using a generated mock from a producer-defined interface leads to imported interface contracts that are too large. Instead, define a minimal interface in the test file or use a function type.

### 5.3 Error Handling Fumbles

- **Logging and returning an error:**
  ```go
  if err != nil {
      log.Printf("failed to save: %v", err)
      return err
  }
  ```
  This duplicates logging up the stack, creating noise. Either handle (log) or return, rarely both.

- **Ignoring errors with `_`:**
  ```go
  body, _ := ioutil.ReadAll(resp.Body)
  ```
  Always check the error.

- **Panic for recoverable errors:** Using `panic` on an invalid user input or a missing file forces the caller to `recover`, which is slow and obscures control flow. Use `error`.

- **Sentinel errors with no context:** `if err == ErrNotFound` is fragile. Use `errors.Is` and wrap errors with `fmt.Errorf("search user %s: %w", id, err)`.

### 5.4 Concurrency Missteps

- **Goroutine leak via unclosed channel or missing context:**
  ```go
  go func() {
      for msg := range ch { // blocks forever if ch is never closed
          process(msg)
      }
  }()
  ```
  Always ensure the sending side closes the channel or the goroutine exits via `ctx.Done()`.

- **Using `time.Sleep` for synchronization:** Sleeping to wait for a goroutine is racy. Use channels, `sync.WaitGroup`, or `errgroup`.

- **Copying a `sync.Mutex`:** If a struct containing a mutex is passed by value after locking, the lock state is copied and subsequent locks on the copy have no effect. Use `go vet` and pass by pointer.

- **Unbuffered channel as a queue in tight loops:** Unbuffered channels enforce synchronous handoff; they’re not an asynchronous queue. For decoupling, use a buffered channel with care or a custom ring buffer if needed, but beware of backpressure.

### 5.5 Package and Naming Woes

- **Overpackaging:** Dozens of packages each with a single file (`types.go`, `interfaces.go`, `errors.go`). This creates complex import graphs and hides cohesion. Co-locate related code in a single package until it grows enough to warrant a clear API boundary.

- **Generic utility packages:** `util`, `common`, `helpers`. These attract miscellaneous functions and kill discoverability. Name packages after the service they provide: `retry`, `textutil`, `pool`.

- **Stuttering names:** `user.UserService` or `user.UserRepository`. Inside the `user` package, just `Service` is clear because the caller sees `user.Service`.

- **Unexported constructors returning exported types:** `func newConfig() *Config` is confusing; exported functions return exported types, unexported may return both but usually for internal wiring.

### 5.6 The `init()` Tangle

- **Registering database drivers or HTTP handlers in `init()`:**
  ```go
  func init() {
      sql.Register("mydriver", &Driver{})
  }
  ```
  This works for plugins, but for application code it couples registration to import, making it impossible to run a process without the side effect. Prefer explicit registration in `main`.

- **Setting global state in `init()`:** Using `init()` to read environment variables and set package-level globals makes testing and configuration hard. Accept configuration explicitly via a struct.

These mistakes stem from bending Go to fit a preconceived architecture. Recognizing them is the first step toward idiomatic clarity.

---

## 6. Performance Considerations

Anti-patterns often carry a hidden performance cost. While Go’s performance is good enough for many applications, the cumulative effect of unnecessary indirection and allocation can become the bottleneck.

**Interface boxing overhead.** Returning an interface from a constructor forces the concrete value to be boxed. Consider a simple `Counter` struct:

```go
type Counter struct{ n int }
func (c *Counter) Inc() { c.n++ }
func (c *Counter) Val() int { return c.n }

func NewCounter() interface{ Inc(); Val() int } {
    return &Counter{}
}
```

Every call to `NewCounter` allocates the `*Counter` on the heap because it escapes through the interface return. Benchmark shows ~40ns/op and 1 alloc/op. If we return `*Counter` directly, escape analysis often keeps it on the stack (if used locally), reducing allocs to zero and cutting latency dramatically. Prefer returning concrete types.

**Excessive use of reflection.** Anti-patterns that use `reflect` for “flexible” configuration or serialization suffer slow path (hundreds of ns vs. tens) and heap allocations. For example, a custom option setter using reflection to call `SetX` methods on a struct is far slower than generated code or manual assignment. The standard library’s JSON encoder uses reflection and can be a bottleneck; you replace it with easyproto or code generation. Avoid reflection in hot paths; use code generation (`go:generate`) for repetitive boilerplate.

**Goroutine spawning without bound.** A pattern that starts a goroutine per incoming item with no concurrency limit can create hundreds of thousands of goroutines under load. While goroutines are cheap (a few KiB), context switching overhead and memory can saturate the scheduler. A worker pool or a semaphore (buffered channel) limits parallelism and reduces GC pressure from stack reallocation.

**Unnecessary pointers and heap allocations.** Java developers tend to make all struct fields pointers. A struct like `type Record struct { ID *int; Name *string; ... }` causes many tiny heap allocations. If zero values are meaningful (`0` for ID, empty string), use value types. The GC has to scan every pointer field. Reducing pointer fields tightens cache locality and lowers GC mark time.

**Copying large data in tight loops.** Using value receivers for methods on huge structs (e.g., a 1KB config passed by value) can be slower than a pointer receiver. Conversely, using a pointer receiver for a small struct that doesn’t need mutation incurs allocation and dereference cost. Measure with benchmarks; there’s no universal rule, but the anti-pattern is ignoring the cost entirely.

**Channel misuse.** Sending on an unbuffered channel in a hot loop serializes goroutines, potentially underutilizing CPU. A common anti-pattern is using a channel as a simple lock-free queue without considering that a channel is a synchronization primitive, not a high-performance data structure. For high-throughput pipelines, use buffered channels or specialized lock-free queues only after profiling.

The performance tax of anti-patterns is often invisible until production. Profiling with `pprof` will reveal these hidden costs, but internalizing the idioms eliminates them by design.

---

## 7. Best Practices

The antidote to anti-patterns is a set of heuristics that produce code that “looks like Go.”

**Accept interfaces, return structs.** Your public functions should accept the smallest interface needed and return concrete types (usually a pointer to a struct, or a value if it’s a small immutable type). This enables the caller to pass a mock or a subset, while the return type remains concrete for the compiler.

**Define interfaces at the consumer.** If you’re writing a package that calls a method, declare the interface with only that method right before the function that uses it. Don’t export the interface unless it’s a well-known abstraction like `io.Reader`.

**Keep interfaces tiny.** Single-method interfaces are the norm (`io.Reader`, `io.Writer`, `http.Handler`). Compose them if you need more. A 5-method interface is a code smell; split it into cohesive roles.

**Use constructor functions that return concrete types, and accept options via functional options.**
```go
type Server struct { ... }
func NewServer(addr string, opts ...Option) *Server { ... }
```

**Error handling with context.** Always wrap errors at the boundary where context is added:
```go
user, err := repo.FindByID(ctx, id)
if err != nil {
    return fmt.Errorf("fetching user %s: %w", id, err)
}
```
Use `errors.Is` and `errors.As` in callers; never type-assert on an error type.

**Zero values are useful.** A `sync.Mutex` is ready to use zero-valued. A `bytes.Buffer` works without initialization. Don’t write `NewBuffer` when `var buf bytes.Buffer` suffices. Make zero values of your own structs usable without a constructor if possible.

**Explicit dependency passing.** Wire dependencies in `main` or a “compose” layer; pass them as arguments. Avoid global state. For large programs, simple dependency injection without a framework (manual `Config` struct with functions) scales surprisingly well.

**Goroutine lifecycles.** Always ask: who owns this goroutine? How does it stop? Use `ctx` for cancellation and `sync.WaitGroup` for join. Encapsulate goroutine management in a `Start`/`Stop` pattern if you must, but prefer synchronous functions that callers launch themselves.

**Package organization.** Name packages for what they provide, not what they contain. A single package with a handful of well-named files beats 10 tiny packages with internal interfaces. Use `internal` to enforce boundaries.

**Leverage `go vet` and static analysis.** Many anti-patterns (copylocks, unreachable code, printf format mismatches) are caught by `go vet`. Integrate it into your CI. Extra linters like `staticcheck` and `golangci-lint` with appropriate rules can detect over-complex interfaces, unused parameters, and more.

**Simplicity as a decision tool.** When faced with a design choice, ask: “Is this the simplest thing that works?” If you’re adding an interface, a factory, or a channel that isn’t strictly necessary, pause. The “Go way” is to defer abstraction until the need is proven.

Following these practices eliminates most anti-patterns before they reach code review.

---

## 8. Examples

Let’s walk through a real refactoring: a small service that retrieves and transforms user data. The initial version is a classic Java-style anti-pattern stew. Then we rewrite it idiomatically.

### Anti-Pattern Code

```go
// package user
package user

import (
    "database/sql"
    "log"
    "errors"
)

// User represents a user entity.
type User struct {
    ID    int
    Name  string
    Email string
}

// UserRepository is the data access interface.
type UserRepository interface {
    FindByID(id int) (*User, error)
    Save(u *User) error
}

// PostgresUserRepo implements UserRepository.
type PostgresUserRepo struct {
    db *sql.DB
}

func NewPostgresUserRepo(db *sql.DB) UserRepository {
    return &PostgresUserRepo{db: db}
}

func (r *PostgresUserRepo) FindByID(id int) (*User, error) {
    u := &User{}
    err := r.db.QueryRow("SELECT id, name, email FROM users WHERE id=$1", id).Scan(&u.ID, &u.Name, &u.Email)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, errors.New("user not found")
        }
        return nil, err
    }
    return u, nil
}

// UserService interface for business logic.
type UserService interface {
    GetUser(id int) (*User, error)
}

type userService struct {
    repo   UserRepository
    logger *log.Logger
}

func NewUserService(repo UserRepository, logger *log.Logger) UserService {
    return &userService{repo: repo, logger: logger}
}

func (s *userService) GetUser(id int) (*User, error) {
    s.logger.Printf("Fetching user %d", id)
    u, err := s.repo.FindByID(id)
    if err != nil {
        s.logger.Printf("Error fetching user: %v", err)
        return nil, err
    }
    return u, nil
}
```

Problems:
- Producer-side interfaces `UserRepository` and `UserService` that force the caller to accept the full contract.
- Concrete implementation `PostgresUserRepo` hidden behind an interface return.
- `UserService` wraps a single function in a struct and interface, adding indirection.
- Logging and returning the error duplicates messages.
- Uses `log` instead of structured logging, but that’s minor.
- `NewPostgresUserRepo` returns an interface, preventing compiler inlining and escape analysis optimization.

### Idiomatic Refactor

```go
// Package user provides user data access.
package user

import (
    "context"
    "database/sql"
    "fmt"
    "log/slog"
)

// User is a core domain type.
type User struct {
    ID    int
    Name  string
    Email string
}

// FindUserByID fetches a user from the database.
// It accepts the concrete DB handle directly; the caller may wrap if needed.
func FindUserByID(ctx context.Context, db *sql.DB, id int) (*User, error) {
    u := &User{}
    err := db.QueryRowContext(ctx,
        "SELECT id, name, email FROM users WHERE id=$1", id).
        Scan(&u.ID, &u.Name, &u.Email)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, fmt.Errorf("user %d: %w", id, ErrNotFound)
        }
        return nil, fmt.Errorf("querying user %d: %w", id, err)
    }
    return u, nil
}

// ErrNotFound is a sentinel error that consumers can check.
var ErrNotFound = fmt.Errorf("user not found")

// --- Higher-level business logic stays in its own package or a simple function. ---

// GetUserInfo returns formatted user info. It accepts exactly what it needs.
// The consumer can define a smaller interface if they want to mock.
type userFinder interface {
    FindByID(ctx context.Context, id int) (*User, error)
}

func GetUserInfo(ctx context.Context, finder userFinder, id int) (string, error) {
    slog.InfoContext(ctx, "Fetching user info", "id", id)
    u, err := finder.FindByID(ctx, id)
    if err != nil {
        return "", fmt.Errorf("GetUserInfo: %w", err)
    }
    return fmt.Sprintf("%s <%s>", u.Name, u.Email), nil
}
```

Key improvements:
- `FindUserByID` is a package-level function that takes a concrete `*sql.DB`. No repository interface.
- `ErrNotFound` is a sentinel error, easily checked with `errors.Is`.
- `GetUserInfo` defines a tiny `userFinder` interface just for the one method it calls. This is consumer-side.
- No `Service` struct, no constructor. Dependencies passed directly.
- Structured logging with context propagation.
- The caller in `main` would wire `db` and call `FindUserByID`, or if they want to test `GetUserInfo`, they implement the 1-method interface.

This refactor is shorter, more explicit, and achieves the same behavior with fewer allocations and indirections.

---

## 9. Summary & Exercises

**Summary**
Anti-patterns in Go typically arise when we import patterns from languages with heavier abstraction norms. The most harmful are producer-side interfaces, large interfaces, unnecessary constructors, interfaces returned from constructors, goroutine leaks, error handling that both logs and returns, and excessive packaging. These patterns violate Go’s core philosophy of simplicity, composition, and explicit dependency management. They also have tangible costs: increased heap allocations, GC pressure, binary bloat, and reduced testability. The antidote is a disciplined return to concrete types, consumer-defined small interfaces, direct error wrapping, and lifecycle-aware concurrency. When you feel the urge to add an abstraction layer, first write the simplest code that works and let the abstraction emerge from real usage.

**Exercises**

1. **Refactor a tangled service.** Given the following code snippet, identify at least three anti-patterns and rewrite the snippet in idiomatic Go. The snippet defines a `ConfigLoader` interface next to its implementation, uses a constructor that returns the interface, and does not handle a file-not-found error gracefully.
   ```go
   type ConfigLoader interface {
       Load(path string) (*Config, error)
   }
   type FileConfigLoader struct{}
   func NewFileConfigLoader() ConfigLoader { return &FileConfigLoader{} }
   func (l *FileConfigLoader) Load(path string) (*Config, error) {
       data, _ := os.ReadFile(path) // error ignored
       // unmarshal...
       return cfg, nil
   }
   ```

2. **Build a thread-safe cache with an appropriate interface.** Design a cache package that stores items with an expiration. The anti-pattern would be to define a `Cache` interface in the `cache` package and export a `NewCache` returning that interface. Instead, return a concrete `*Cache` and allow consumers to define the methods they need. Write the package and a test that defines a minimal `itemFetcher` interface to mock the “fetch on miss” behavior. Ensure your implementation handles context cancellation, uses proper mutex locking, and avoids goroutine leaks.

3. **Detect and fix concurrency anti-patterns.** The following code starts a goroutine in a constructor that reads from a channel, but the channel is never closed. It also uses a global logger from `init()`. Rewrite it so that the goroutine is stoppable via a context, the channel is properly closed, and the logger is passed as an explicit dependency. Use the `slog` package.
   ```go
   package worker

   import "log"

   var logger *log.Logger

   func init() {
       logger = log.Default()
   }

   type Worker struct {
       ch chan int
   }

   func NewWorker() *Worker {
       w := &Worker{ch: make(chan int)}
       go func() {
           for v := range w.ch {
               logger.Printf("got %d", v)
           }
       }()
       return w
   }
   ```

These exercises challenge you to internalize the idiomatic reflexes and reject the anti-patterns that slip in when you’re not looking. The goal is not perfection but a permanent shift toward Go’s philosophy: solve the problem, not the abstraction.
