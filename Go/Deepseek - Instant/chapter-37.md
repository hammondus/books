# Chapter 37: Architecture & Project Structure

## 1. Basic Usage

Go’s standard library and toolchain impose almost no requirements on how you organise your source files beyond the rules of package visibility and import paths. This freedom is a double‑edged sword. For a seasoned engineer, the first decision when starting a Go project is not “which framework do I adopt?” but “how do I arrange my directories and packages so that the system remains understandable, testable, and scalable?”

The most minimal project—a single‑file command‑line tool—is trivial:

```go
// cli-tool/main.go
package main

import (
    "fmt"
    "os"
)

func main() {
    if len(os.Args) < 2 {
        fmt.Fprintln(os.Stderr, "usage: cli-tool <name>")
        os.Exit(1)
    }
    fmt.Printf("Hello, %s\n", os.Args[1])
}
```

Build with `go build -o cli-tool .` and run. No further structure required.

For a production‑grade web service, the idiomatic pattern is to separate **domain logic**, **adapters** (HTTP handlers, database repositories), and **configuration/wiring**. The following layout has become a de facto standard (though not official):

```
myapp/
├── cmd/
│   └── server/
│       └── main.go                # wire everything together, start server
├── internal/
│   ├── config/                    # configuration loading & validation
│   ├── domain/                    # core business logic (no dependencies)
│   └── service/                   # orchestration layer
├── pkg/                           # public API that other projects may import
│   └── httpclient/                # reusable HTTP client wrapper
├── api/                           # OpenAPI/protobuf definitions
├── migrations/                    # database schema migrations
└── go.mod
```

**Dependency injection in Go** is almost always done manually, without magic containers. You pass concrete dependencies via constructors, often returning interfaces to ease testing:

```go
// internal/domain/weather.go
package domain

type WeatherService interface {
    GetTemperature(ctx context.Context, city string) (float64, error)
}

// internal/service/forecast.go
package service

import "myapp/internal/domain"

type ForecastService struct {
    weather domain.WeatherService
    cache   *Cache
}

func NewForecastService(w domain.WeatherService, cache *Cache) *ForecastService {
    return &ForecastService{weather: w, cache: cache}
}

func (f *ForecastService) Today(ctx context.Context, city string) (string, error) {
    temp, err := f.weather.GetTemperature(ctx, city)
    if err != nil {
        return "", err
    }
    return fmt.Sprintf("It's %.1f°C", temp), nil
}
```

Wiring happens in `cmd/server/main.go`:

```go
package main

import (
    "myapp/internal/config"
    "myapp/internal/domain"
    "myapp/internal/service"
    "myapp/pkg/weatherclient"
)

func main() {
    cfg := config.Load()
    // concrete implementation of domain.WeatherService
    weatherClient := weatherclient.New(cfg.WeatherAPIKey)
    cache := service.NewCache(5*time.Minute)

    forecastSvc := service.NewForecastService(weatherClient, cache)
    // ... attach handlers and start server
}
```

This pattern gives you full control over lifetimes, easy mocking in tests (you can pass any implementation of `domain.WeatherService`), and zero runtime reflection overhead.

## 2. Under the Hood

Go’s compilation model and visibility rules are the foundation on which every project structure is built.

**Package as the unit of compilation.** When you run `go build`, the compiler processes all `.go` files inside a package together. Identifiers starting with an uppercase letter are **exported** (visible to importers); lowercase identifiers are **unexported** (private to the package). There are no “sub‑packages” in the sense of inheritance—a package `internal/db` is just a separate package whose import path happens to contain slashes.

**Internal directories.** The `internal/` directory is a compiler‑enforced privacy boundary: code inside `internal/` can only be imported by packages that are inside the same module and are **descendants** of the `internal/` directory’s parent. For example:

```
myapp/
├── internal/
│   └── auth/
│       └── token.go
├── cmd/
│   └── server/
│       └── main.go          // can import "myapp/internal/auth"
└── pkg/
    └── client/
        └── client.go        // cannot import "myapp/internal/auth"
```

This is **not** a security feature (anyone can still read the source code), but a design tool that signals “this package is not part of your public API.” The compiler prevents accidental leakage.

**Init functions.** Each package can define zero or more `func init()`. They run in a deterministic order (after all variable declarations are evaluated, and after imported packages’ `init` functions). Many structured projects abuse `init()` for registration (e.g., automatically adding routes to a global router). This creates hidden dependencies and makes testing difficult. The Go philosophy prefers **explicit wiring in `main`**.

**Build constraints and tags.** While not directly about architecture, build tags (`//go:build integration`) let you split test suites or platform‑specific code. A well‑structured project often places platform‑dependent implementations (e.g., Windows vs Linux) into separate files with build tags, keeping the core logic clean.

**No module‑level cyclic imports.** Go strictly forbids import cycles at compile time. This is a **blessing in disguise**: it forces you to keep dependency graphs acyclic. In practice, cycles are resolved by introducing an interface in a shared low‑level package or by restructuring layers (e.g., moving the offending code into a new package that both previous packages can depend on).

## 3. Why This Design?

Go’s lack of a mandated architecture is not an omission—it is a **deliberate choice** rooted in the language’s “less is more” philosophy. The Go team observed that frameworks (Spring, Rails, Django) impose **heavy conventions that become liabilities** as projects age. By giving engineers only packages and visibility, Go forces you to think about coupling and cohesion without paying the cost of a meta‑framework.

**Why no built‑in dependency injection container?**  
Languages like Java (Spring) and C# (ASP.NET) use reflection‑based containers that inject dependencies automatically based on annotations or conventions. This approach reduces boilerplate but:
- Moves control flow away from explicit code → harder to reason about startup.
- Hides performance costs (reflection, proxy generation).
- Complicates debugging (stack traces become framework‑opaque).

Go’s designers argue that **manual dependency injection** (constructors that accept interfaces, called “wiring code”) is simpler, faster, and more debuggable. The extra 30 lines in `main.go` are a small price for clarity.

**Why no standard project layout in the toolchain?**  
The `go` command does not enforce a `src/`, `lib/`, or `bin/` structure (unlike `GOPATH` days). This freedom acknowledges that **projects have different lifecycles**: a library, a CLI, a microservice, and a monorepo all benefit from different layouts. Imposing one canonical structure would hurt innovation and create friction for existing tooling (e.g., Bazel, Nix).

**The internal/ convention** was added in Go 1.4 after community feedback. Before that, there was no way to mark a package as “internal to this module.” The design is intentionally minimal: it’s just a reserved directory name that the compiler treats specially. No extra configuration files, no inheritance.

**Compare with Java’s package‑by‑layer vs. package‑by‑feature.**  
Java projects often group by technical layer (`com.example.controller`, `com.example.service`, `com.example.repository`). Go’s community discovered that **package‑by‑feature** (`internal/user`, `internal/order`) reduces coupling: changes to a feature rarely force changes in unrelated packages. Go’s short import paths (`user`, `order`) encourage this style.

## 4. Competing Approaches

| Language / Ecosystem | Typical Structure | DI Mechanism | Trade‑offs |
|----------------------|------------------|--------------|-------------|
| **Java (Spring Boot)** | Package by layer (`controller`, `service`, `repository`) + Maven/Gradle modules | Annotation‑driven (`@Autowired`, `@Component`) | High startup cost, magic, but excellent for large teams. |
| **Java (Plain)** | Similar layering, but manual instantiation with `new` | Constructor injection by hand | Boilerplate, but explicit and fast. |
| **Python (Django)** | Apps (each app has `models.py`, `views.py`), project‑wide settings | Global request/response, signals, or manual | “Batteries‑included” but tight coupling to Django. |
| **Python (FastAPI)** | Routers + dependencies via `Depends()` | Callable injection (runtime) | Lightweight, but still relies on runtime introspection. |
| **Node.js (Express)** | No standard; often `routes/`, `controllers/`, `models/` | Manual `require` / `import` | Very flexible, but no compiler help for cycles. |
| **Node.js (NestJS)** | Modules, controllers, providers (Angular‑style) | Reflection + decorators | Rich ecosystem, but complexity similar to Spring. |
| **Rust** | Cargo workspaces, crates. Usually `src/lib.rs` + `src/bin/` | Manual (pass traits) or compile‑time DI (e.g., `dodrio`) | No runtime overhead; ownership complicates DI in some cases. |
| **Go (this chapter)** | `cmd/`, `internal/`, `pkg/` (optional) | Manual constructor injection | Explicit, fast, zero magic. Repetitive for very deep graphs. |

**Why engineers from Java/C# struggle initially** with Go’s structure:
- They miss `@Autowired` and look for a “Spring for Go” (e.g., `wire`, `fx`). Those exist, but the community warns against using them until pain is severe.
- They tend to create deeply nested packages (`internal/service/impl/v2/...`), mimicking Java’s `impl` sub‑packages. In Go, an interface is usually defined in the package that consumes it, not in a separate `iface` package.
- They overuse `pkg/` for everything, forgetting that `pkg/` is meant for **reusable library code**, not internal implementation details.

## 5. Common Mistakes

**1. Overusing `/pkg`**  
The `/pkg` directory is not required. Many successful projects (Docker, Kubernetes, Go itself) do not use it. The mistake is moving every internal package into `pkg/`, expecting it to be a “better” default. `pkg/` should contain code that **other modules may import and depend on**. If you are writing an application, 90% of your code belongs in `internal/` or directly under the root.

**2. The “one huge package” anti‑pattern**  
Dumping everything into `package main` or `package models` leads to:
- Slow compilation (the compiler has to re‑analyse everything on any change).
- Impossible to test in isolation.
- Frequent import cycles as soon as you try to split.

Refactor early: if a file exceeds ~500 lines, consider extracting a sub‑package.

**3. Artificial hierarchies**  
Novices write `internal/utils/stringutil/`, `internal/utils/timeutil/`, `internal/utils/httputil/`. This is **package by kind**, not by feature. Instead, put string helpers inside the package that needs them; if truly shared, promote to `internal/strings` or even `pkg/strings`. Deep nesting adds no value and makes `import` lines longer.

**4. Premature interface extraction**  
The Java habit of defining an interface for every service (e.g., `UserService` interface, then `UserServiceImpl`) leads to pointless indirection. In Go, define an interface **only when you have at least two concrete implementations** (real implementation + mock, or two production variants). Start with a concrete struct; replace with an interface later when the need arises.

**5. Misusing `internal/` for cross‑module boundaries**  
`internal/` is **module‑scoped**, not repository‑scoped. If you have a monorepo with multiple Go modules (each with its own `go.mod`), a package under `internal/` in module A cannot be imported by module B. That’s correct. However, if you want to share code across modules, put it in `pkg/` or create a third shared module.

**6. Circular dependencies caused by `init` functions**  
Imagine package `config` reads environment variables and calls `log.SetPrefix()`. Package `log` imports `config` to read the log level. Now `config` → `log` (via `init`) and `log` → `config` (via import). The compiler rejects this. Solution: move the logging setup to `main` after both packages are initialised.

**7. Over‑engineering “clean architecture” boundaries**  
Drawing four concentric circles (entities, use cases, interface adapters, frameworks) in a 2000‑line service creates massive boilerplate. Go’s strength is being **pragmatic**: a simple `internal/` with handlers, services, and a repository package is often sufficient for years.

## 6. Performance Considerations

Project structure affects performance in subtle but real ways:

- **Compilation speed.** The Go compiler works in parallel across packages. Splitting a monolithic package into several smaller ones **speeds up incremental builds** because only changed packages are rebuilt. However, too many packages (hundreds) increase the overhead of reading export data and type‑checking. A good balance is 10–50 packages for a typical microservice.

- **Binary size and dead code elimination.** The linker performs dead code elimination at the **package and function level**, not the “class” level. If you import a large package but only use one function, the entire package is still linked unless you use `-ldflags=-s` (which strips debug info but not unused functions). Splitting a utility package into several smaller packages (`strutil`, `timeutil`, `httputil`) can reduce binary size, because you only import the parts you need.

- **Inlining decisions.** The compiler inlines small functions (generally ≤~80 bytes of IR) only within the same package or across packages when the function is marked `//go:noinline`? Actually, cross‑package inlining is possible **if the function body is available** – that requires the target package to be compiled with `-l=2` (the default). But splitting a function across many packages does **not** prevent inlining; the compiler will inline if the function is small and marked as such. However, virtual calls via interfaces cannot be inlined (except with PGO in Go 1.21+). Excessive interface abstraction therefore **can** harm performance, which is another argument against premature interface extraction.

- **Escape analysis and heap allocations.** The compiler’s escape analysis works within a package; inter‑package analysis is limited because the compiler sees only the caller’s side plus the callee’s export signature. If you pass a pointer from package A to a function in package B, and B stores that pointer into a global variable, the escape analysis may still decide that the pointer escapes to heap even if B does nothing. This is rarely a reason to avoid splitting packages, but be aware that heavy pointer passing across package boundaries can hinder optimisation.

- **Startup time.** If you abuse `init()` functions across many packages (e.g., each package registers something with a global registry), startup time increases linearly with the number of packages. The worst offenders are database drivers (`_ "github.com/lib/pq"`) that register themselves. Keep `init()` trivial – ideally just `init() {}` or validation of static data.

## 7. Best Practices

The Go community has converged on a set of **idiomatic architectural guidelines** over the past decade. Follow these unless you have a compelling reason not to.

### Project Layout (the “Standard” one – with caveats)

The widely referenced [golang-standards/project-layout](https://github.com/golang-standards/project-layout) is **not official** but is battle‑tested. Use its core ideas:

- `/cmd` – one subdirectory per executable (e.g., `/cmd/server`, `/cmd/cli`). Keep each `main.go` tiny (only wiring and flag parsing).
- `/internal` – private code that no external module should import. Sub‑packages under `internal/` follow the same rules.
- `/pkg` – **optional**. Only for library code that is safe for external consumption. Many projects skip this and put public API directly under root (e.g., `github.com/gorilla/mux` has top‑level `mux.go`).
- `/api` – API contract definitions (OpenAPI, Protocol Buffers, Thrift).
- `/web` – static web assets (if any).
- `/configs` – configuration file templates (not runtime config).
- `/scripts` – build/deploy scripts.

**Crucially:** Do **not** put `src/` directory. Do **not** nest `internal/` inside `pkg/`. Do **not** create `lib/` or `common/` – those become dumping grounds.

### Package Design – “Package by Feature, Not by Layer”

Instead of:

```
internal/
├── handlers/
│   ├── user.go
│   └── order.go
├── services/
│   ├── user.go
│   └── order.go
└── repositories/
    ├── user.go
    └── order.go
```

Prefer:

```
internal/
├── user/
│   ├── handler.go      (HTTP or gRPC transport)
│   ├── service.go      (business logic)
│   └── repository.go   (database interface + implementation)
├── order/
│   ├── handler.go
│   ├── service.go
│   └── repository.go
```

Benefits:
- When you change a feature, you touch only one package (or a few).
- You can delete a feature by removing its directory.
- Dependencies are explicit: `order` may import `user` for user data, but not the reverse.

### Dependency Injection – Manual & Constructor‑Based

- Every major dependency should be passed as a parameter to the constructor (`NewX(...)`).
- Return concrete types from constructors, unless you need an interface immediately (e.g., for testing).
- Assemble the object graph in `main()` (or in a `wire.go` if using `wire`). This is called the “dependency root”.
- Avoid global variables (except for truly immutable, process‑wide constants like `version`).

**Example of a well‑wired main:**

```go
func main() {
    cfg := config.MustLoad()

    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: cfg.LogLevel,
    }))

    db := postgres.MustOpen(cfg.DatabaseURL)
    userRepo := postgres.NewUserRepository(db)
    userService := user.NewService(userRepo, logger)

    orderRepo := postgres.NewOrderRepository(db)
    orderService := order.NewService(orderRepo, userService, logger)

    srv := server.New(cfg.ListenAddr, userService, orderService, logger)
    srv.Run()
}
```

### When to Use Interfaces

- **In the consuming package:** Define an interface for the behaviour you need. Example: `user/service.go` defines `type Repository interface { ... }`. The concrete implementation lives in `postgres/user_repo.go` or `memory/user_repo.go`.
- **To decouple packages:** If `order` needs to call `user` to check permissions, define a small interface in the `order` package: `type UserChecker interface { CanPlaceOrder(ctx context.Context, userID string) bool }`. Then `userService` implements that interface. This keeps `order` from importing the entire `user` package.

### Use `internal/` Liberally for Application Code

Any package that is not meant to be imported by another module should be placed under `/internal`. This prevents accidental API breakage. Even helper packages like `internal/httputil` belong there – they are an implementation detail.

### Keep `go.mod` Clean

Run `go mod tidy` regularly. Use `go work` for multi‑module development (e.g., monorepo with several microservices). Do not commit `vendor/` unless you have a specific offline build requirement.

### Documentation

Place a `doc.go` file in every non‑trivial package to explain its purpose and high‑level usage. Example:

```go
// Package user provides user management (registration, authentication,
// profile updates). It depends on a Repository interface that must
// be supplied by the caller.
package user
```

This appears in `go doc` and `pkg.go.dev`.

## 8. Examples

### Example 1: Complete HTTP service with clean separation

```go
// internal/todo/todo.go
package todo

import "context"

type Item struct {
    ID   string
    Text string
    Done bool
}

type Repository interface {
    Save(ctx context.Context, item *Item) error
    FindByID(ctx context.Context, id string) (*Item, error)
}

type Service struct {
    repo Repository
}

func NewService(repo Repository) *Service {
    return &Service{repo: repo}
}

func (s *Service) MarkDone(ctx context.Context, id string) error {
    item, err := s.repo.FindByID(ctx, id)
    if err != nil {
        return err
    }
    item.Done = true
    return s.repo.Save(ctx, item)
}
```

```go
// internal/todo/http_handler.go
package todo

import (
    "encoding/json"
    "net/http"
)

type HTTPHandler struct {
    service *Service
}

func NewHTTPHandler(svc *Service) *HTTPHandler {
    return &HTTPHandler{service: svc}
}

func (h *HTTPHandler) MarkDone(w http.ResponseWriter, r *http.Request) {
    var req struct {
        ID string `json:"id"`
    }
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    if err := h.service.MarkDone(r.Context(), req.ID); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    w.WriteHeader(http.StatusNoContent)
}
```

```go
// internal/memory/todo_repo.go (implements todo.Repository)
package memory

import (
    "context"
    "sync"
    "myapp/internal/todo"
)

type TodoRepository struct {
    mu    sync.RWMutex
    items map[string]*todo.Item
}

func NewTodoRepository() *TodoRepository {
    return &TodoRepository{items: make(map[string]*todo.Item)}
}

func (r *TodoRepository) Save(ctx context.Context, item *todo.Item) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.items[item.ID] = item
    return nil
}

func (r *TodoRepository) FindByID(ctx context.Context, id string) (*todo.Item, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    item, ok := r.items[id]
    if !ok {
        return nil, fmt.Errorf("item %s not found", id)
    }
    return item, nil
}
```

```go
// cmd/server/main.go
package main

import (
    "log"
    "net/http"
    "myapp/internal/todo"
    "myapp/internal/memory"
)

func main() {
    repo := memory.NewTodoRepository()
    svc := todo.NewService(repo)
    handler := todo.NewHTTPHandler(svc)

    http.HandleFunc("/todo/done", handler.MarkDone)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Example 2: Library‑style project with reusable `pkg/`

```go
// pkg/retry/retry.go
package retry

import (
    "context"
    "time"
)

type BackoffFunc func(attempt int) time.Duration

func ExponentialBackoff(base time.Duration) BackoffFunc {
    return func(attempt int) time.Duration {
        return base * (1 << attempt)
    }
}

func Do(ctx context.Context, maxAttempts int, backoff BackoffFunc, fn func() error) error {
    var err error
    for attempt := 0; attempt < maxAttempts; attempt++ {
        if err = fn(); err == nil {
            return nil
        }
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(backoff(attempt)):
        }
    }
    return err
}
```

Any external project can `import "github.com/you/pkg/retry"`. The `pkg/` name signals that this package is **stable and supported** for external use.

## 9. Summary & Exercises

### Summary

- Go’s architecture is **not prescribed** by the language; you must design it thoughtfully.
- The `cmd/`, `internal/`, `pkg/` layout is a battle‑tested starting point for applications.
- **Manual dependency injection** via constructors is the idiomatic Go way – no reflection‑based containers.
- **Package by feature** reduces coupling and improves compile‑time parallelism.
- Use `internal/` to hide implementation details from external modules.
- Prefer **small interfaces** defined in the consumer package, not in a shared “interfaces” package.
- Over‑engineering (e.g., Clean Architecture for a 2000‑line service) is a common mistake – start simple.

### Exercises

**Exercise 1 – Refactor a monolithic CLI**  
Take the following monolithic CLI code (single `main.go` with 400 lines). Identify three distinct responsibilities (e.g., configuration, HTTP client, output formatting). Split it into a `cmd/cli/main.go`, an `internal/config` package, an `internal/fetcher` package, and an `internal/display` package. Use dependency injection to wire them together in `main`. Verify that `go test ./...` still passes (write minimal tests for each package).

**Exercise 2 – Build a manual DI container (to understand trade‑offs)**  
Write a small package `di` that uses a map of types (via `reflect.Type`) to store singletons. Provide `Provide[T any](constructor func() T)` and `Invoke[T any]() T` functions that resolve dependencies by examining constructor parameters. Add a cyclic dependency detection. Compare the resulting code (in a toy application with 5–10 services) against manual wiring. Write a short analysis of the readability, debuggability, and performance differences.

**Exercise 3 – Design a project structure for a multi‑command daemon**  
You are building a monitoring agent that can run in three modes: `agent run` (long‑running collector), `agent once` (single collection cycle then exit), and `agent test` (run self‑tests). The agent collects metrics from plugins (CPU, memory, disk). Design a directory and package layout that allows all three commands to share the core collection logic but keep mode‑specific code separate. Show the `main.go` for each command (using `cobra` or a simple flag switch). Explain where interfaces between the core and the plugins should be defined.
