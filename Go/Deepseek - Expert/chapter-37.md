# 37. Architecture & Project Structure

Go’s core philosophy—simplicity, composition, explicitness—naturally extends into how we lay out codebases. This chapter explores the idioms and trade-offs of structuring Go programs, from a single `main.go` to large monorepos, and how architectural patterns like Dependency Injection and Hexagonal Architecture map onto Go’s type system. Crucially, we’ll draw a line between sensible architecture and over-engineering, because in Go, the best architecture is often the one you didn’t have to build.

## Basic Usage

Go projects typically grow from a single `main.go` file into a multi-package tree. The de facto community standard, often called the **Go project layout**, recommends a set of well-known directories. It’s not mandatory—the Go team explicitly does not endorse a single layout—but it has become a lingua franca for tooling and human expectations.

```text
myproject/
├── cmd/                  # Application entry points
│   └── myapp/
│       └── main.go
├── internal/             # Private application code (not importable by external modules)
│   ├── service/
│   │   └── user.go
│   └── repository/
│       └── user_repo.go
├── pkg/                  # Library code that can be imported by external consumers
│   └── auth/
│       └── tokens.go
├── api/                  # gRPC / OpenAPI / Protobuf definitions
├── web/                  # Static assets, templates
├── scripts/              # Build, install, analysis scripts
├── configs/              # Configuration file templates
├── go.mod
└── go.sum
```

- `cmd/`: each subdirectory holds a `main` package that produces an executable. Keep it thin—just parse flags, wire dependencies, and run.
- `internal/`: enforces an import boundary. The Go toolchain prevents any module outside the current module’s tree from importing packages rooted at `internal`. This is your main weapon for encapsulation.
- `pkg/`: shared libraries meant for external reuse. Use sparingly; most code belongs in `internal` until you have proven it needs to be public.

In a monorepo, you might see a layout like:

```text
monorepo/
├── libs/
│   ├── db/
│   │   └── ...          # shared database utilities
│   └── logging/
│       └── ...          # shared structured logging helper
├── services/
│   ├── auth/
│   │   ├── cmd/auth/
│   │   │   └── main.go
│   │   ├── internal/
│   │   │   ├── service/
│   │   │   └── repository/
│   │   └── go.mod       # optional per-service module
│   └── inventory/
│       └── ...
└── go.work               # Go workspace file (multi-module workspaces)
```

With **Go workspaces** (introduced in Go 1.18), you can work on multiple modules simultaneously without `replace` directives. A `go.work` file lists the module paths, and `go build` automatically resolves local dependencies. This is a game changer for monorepos.

**Dependency Injection (DI)** in Go is simply struct fields and constructor functions. There are no annotations, no container runtime, no reflection-based wiring. You pass dependencies explicitly.

```go
// service/user.go
type UserService struct {
    repo UserRepository
    auth AuthClient
    log  *slog.Logger
}

func NewUserService(repo UserRepository, auth AuthClient, log *slog.Logger) *UserService {
    return &UserService{repo: repo, auth: auth, log: log}
}
```

That’s it. At the composition root (usually `main` or a dedicated `wire` package), you instantiate concrete types and inject them.

```go
func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
    db, _ := sql.Open("postgres", dsn)
    userRepo := repository.NewPostgresUserRepo(db, logger)
    authClient := auth.NewOIDCClient(cfg.AuthURL, logger)
    userSvc := service.NewUserService(userRepo, authClient, logger)
    // ...
}
```

If the wiring becomes complex, tools like **Wire** (from Google) generate the initialization code from provider functions, but many teams find explicit `main` wiring sufficient even for large systems.

**Hexagonal (Ports & Adapters) Architecture** maps naturally to Go’s interfaces. The “port” is a Go interface defined by the core business logic. The “adapter” is a struct that implements that interface using a specific technology (Postgres, Redis, HTTP, gRPC).

```go
// internal/core/ports.go
type UserStore interface {
    GetByID(ctx context.Context, id string) (User, error)
}

// internal/adapters/postgres_store.go
type PostgresUserStore struct { db *sql.DB }
func (s *PostgresUserStore) GetByID(ctx context.Context, id string) (User, error) { ... }
```

Business logic depends only on interfaces, never on adapter implementations.

## Under the Hood

The structure of a Go program directly impacts compilation speed, binary size, and even runtime initialization order.

**Package initialization** follows a depth-first, left-to-right dependency order. All `init()` functions in a package run, then its imported packages, then the importing package. Circular imports are a compile-time error. This forces a **directed acyclic graph** of dependencies—no cycles. Architecture decisions must respect that graph.

The **import graph** determines what the compiler processes. A monorepo with many packages but few actual dependencies compiles faster because the compiler only visits what is imported. However, if every package imports a massive `common` utility package, you pay the cost of building `common` for everything. The `internal` boundary prevents external consumers from pulling in that weight, but inside the monorepo, careless imports can still bloat build times.

**Workspace mode** (`go.work`) changes module resolution. When you run `go build` in a workspace, the toolchain uses the local on-disk versions of modules listed in `go.work`, not the remote versions from the proxy. This avoids `replace` directives cluttering `go.mod` and makes working across multiple modules seamless. The workspace file is not committed to version control if the modules are independently published; it’s a local development tool. In a monorepo where modules are never published independently, committing `go.work` can be appropriate.

**Binary size** and **build caching** are affected by how you partition code. A single large package prevents the compiler from caching finer-grained compilation units. Splitting into many small packages can improve incremental build times but may increase binary size due to more exported symbols and relocation entries. The linker does dead code elimination, but only at the function level within a package; unused exported functions from a package may still be included if the package is imported somewhere. The `internal` constraint helps here: unused code in `internal` packages is more likely to be entirely pruned if the importing package is pruned.

## Why This Design?

Go’s approach to project structure is intentionally **minimalist and convention-light**. The Go team famously refused to define an official layout because they view it as a constraint that discourages thinking about what your project actually needs. Instead, they rely on tooling-enforced boundaries (`internal`) and community conventions that emerged from real projects.

**Simplicity over framework**: In Java or C#, dependency injection often requires a framework (Spring, Guice, Unity) that manages object lifetimes and wiring via reflection. Go rejects that. Explicit wiring is not “boilerplate”—it’s an honest display of dependencies. When you have 100 injected fields, you see a problem directly in your constructor signature. Frameworks hide that complexity, delaying the inevitable redesign. Go pushes you to notice and simplify.

**Explicit imports as architecture**: Go’s compiler-enforced acyclic import graph is a design constraint that forces you to think about layering. You can’t accidentally create circular references between packages. This naturally leads to a layered architecture: domain logic in the innermost packages, adapters outside, entry points at the edge. It’s an emergent property, not a rule you have to enforce through static analysis plugins.

**No inheritance, only composition**: Go’s struct embedding and interface satisfaction shape how you structure code. There’s no base class to hold shared configuration; you use embedded structs or shared helper functions. Interfaces are defined at the point of use, not at the point of implementation, which inverts the dependency direction: the consumer says what it needs, and any concrete type that matches can be plugged in. This is the heart of hexagonal architecture—the core defines ports (interfaces), and adapters implement them, often in completely separate packages.

**Workspaces over monorepo tooling**: Instead of requiring a monorepo tool like Bazel or Pants, Go gives you `go.work` that just works with standard Go modules. It’s a simple, composable solution that doesn’t require learning a new build language.

## Competing Approaches

**Java / Spring Boot**: A typical Java service uses a framework-defined project structure (`src/main/java`, `resources`, `test`). Dependency injection is handled by a container; interfaces are often explicitly declared by the implementation class (`implements`). This leads to a proliferation of interfaces paired with a single implementation, often called the “interface-implementation” anti-pattern. In Go, interfaces are defined at the consumer side, drastically reducing boilerplate. Project layout is entirely up to you, which can be liberating or disorienting.

**Python**: Python’s packaging (`pyproject.toml`, `setup.cfg`) and module system are less strict. Circular imports are allowed (but discouraged), and there is no `internal` mechanism. Python projects often rely on namespace packages and conventions (`src` layout) to separate concerns. Dependency injection is often done manually or with frameworks like `dependency-injector`, which use decorators and type hints. Go’s static compilation and explicit wiring avoid runtime surprises.

**Rust**: Rust’s module system with `crate`-level visibility and workspaces is conceptually close to Go’s modules and `internal`. Rust encourages a similar philosophy of small, composable crates. However, Rust adds the complexity of feature flags and conditional compilation, which can lead to intricate crate hierarchies. Go’s simplicity avoids that, at the cost of less flexibility in optional dependencies.

**JavaScript / Node.js**: Monorepo tooling (Lerna, Nx, Turborepo) is common. Dependency injection is often done via dynamic import or factory functions. Go’s static import graph eliminates the category of “require a dependency at runtime” errors, making the structure easier to reason about at scale.

**Microservices vs. Monorepo**: The industry swing between “many repos” and “monorepo” is ongoing. Go does not force either; you can have a repo per service, or a single repo with multiple services. The `go.work` feature makes monorepos more ergonomic. However, a monorepo brings the risk of tight coupling: it’s too easy to import a package from another service and blur the boundary. Strict use of `internal` and code ownership rules mitigates this.

## Common Mistakes

1. **The `pkg` dumping ground**: Beginners often put everything in `pkg/` because they want it to be “reusable.” Reusability should be proven, not assumed. The result is a public API surface that is too large, hard to change, and full of unrelated code. Most code should start in `internal/`. Only move it to `pkg/` when you have multiple consumers outside the current module—or if you are developing a true library.

2. **Over-structured service layers**: A Java developer might create `service/UserService`, `repository/UserRepository`, `model/User`, `dto/UserDTO`, `handler/UserHandler`, and `converter/UserConverter`. In Go, that’s noise. A `user` package containing `Service`, `Store`, and `User` types is often sufficient. Don’t split until you feel real pain from a single file.

3. **Premature hexagonal architecture**: Applying ports & adapters to a tiny CLI tool that fetches a URL and prints it is a waste. Hexagonal architecture pays off when you have multiple adapters (e.g., Postgres, in-memory, and a mock for tests) or you need to swap implementations. If there’s only one database and no testing need for a fake, just use the `sql.DB` directly. Respect **YAGNI**.

4. **Circular import attempts**: When a `service` package needs the `model` package, and the `model` needs something from `service`, you’ve hit a cycle. The fix is often to extract shared types into a separate package (e.g., `core` or `domain`) that both depend on, or to restructure the responsibilities so the dependency is one-way. Don’t work around cycles with `interface{}` (now `any`) just to break the compiler error; that hides the design flaw.

5. **God package**: A single `util` or `common` package that accumulates every helper function. This creates a massive dependency that everything imports, slowing compilation and coupling unrelated parts. Break it into focused packages like `retry`, `timeutil`, `netutil`.

6. **`init()` abuse**: Using `init()` to wire dependencies or register drivers globally (like `database/sql` drivers do) can be acceptable for plugins, but in application code it makes initialization order implicit and hard to test. Prefer explicit initialization in `main`.

7. **Ignoring `internal`**: Without `internal`, any package in your module can be imported by any other package, leading to unintended coupling. Use `internal` to protect your domain logic from being pulled into API handlers or adapters.

## Performance Considerations

Project structure isn’t usually the first place you look for performance gains, but it affects:

**Compilation speed**: The largest cost is parsing and type-checking. A flat structure with a few large packages can be faster to compile than hundreds of tiny packages because there are fewer import edges and less linker work. But incremental builds benefit from many small packages: changing one package only triggers recompilation of its importers. Go’s compiler is fast enough that for most projects (up to a few hundred thousand lines), structure doesn’t dominate build time. Avoid import cycles, and don’t create a package for every struct.

**Binary size**: Unused code elimination works across packages. If you split a `core` package with many functions into many small packages, and a binary only imports a few of them, the linker can omit the others. But exporting symbols (`func Foo`) prevents elimination even if the caller doesn’t use them, if the package is imported. Use unexported helpers where possible, and keep `pkg/` truly minimal.

**Startup time**: `init()` functions run sequentially at program startup. A deep dependency tree can cause noticeable startup latency, particularly if `init()` performs I/O or CPU-intensive work. Structure your code so that heavy initialization is deferred or done lazily.

**GC and allocation**: Not directly structural, but how you design your layers affects allocation. For example, using interfaces for every boundary adds a vtable lookup and potential heap escape for values passed through interface methods. In a hot path, this might matter. The hexagonal architecture’s abstraction layers should be thin; consider measuring if you see excessive allocation from interface boxing.

## Best Practices

1. **Start flat, go deep only when necessary**. A typical new project begins with a few packages: `cmd/server`, `internal/service`, `internal/store`, `internal/handler`. As the domain grows, extract cohesive units into subpackages (e.g., `internal/order`, `internal/user`). Let the package name reflect a domain concept, not a technical layer.

2. **Consumer-defined interfaces**. In hexagonal architecture, the core package defines the interface it needs. The adapter package provides a concrete implementation. This keeps the core free of technology details.

3. **Wire manually, use tools sparingly**. Manual dependency injection is clear and debuggable. If you have dozens of constructors and find the wiring painful, consider whether your system is too granular. Tools like `wire` can help, but they add code generation complexity. Use them only when the wiring genuinely becomes unmanageable.

4. **`internal` as your API boundary**. Everything that is not meant to be consumed by other services or external modules goes into `internal/`. This communicates intent and physically prevents import.

5. **Monorepo with workspaces**: If you have multiple services that share common code, a monorepo with a `go.work` file is ergonomic. Structure it so that each service is a separate module (with its own `go.mod`) if they have independent deployment cycles; otherwise a single module is simpler. Use `internal/` across service boundaries only if you’re okay with a tighter coupling.

6. **Naming conventions**: Package names should be short, lowercase, single word, no underscores. Avoid generic names like `base` or `common`. A package `user` is better than `userservice`. The package path already conveys hierarchy.

7. **Testing structure**: Place test files alongside the code they test (`_test.go`). For integration tests that require building the whole binary, put them in a separate package (often `test` or `integration`) inside the module, or use a `testdata` directory.

8. **Configuration and flags**: Don’t litter flag parsing across packages. Concentrate configuration in `main` or a dedicated `config` package, and pass resolved values into structs.

## Examples

**Example 1: Small CLI tool layout**

```text
mycli/
├── cmd/mycli/main.go
├── internal/
│   ├── fetch/
│   │   └── fetch.go      // logic to fetch data
│   └── output/
│       └── output.go     // formatting output
├── go.mod
└── go.sum
```

`main.go`:

```go
func main() {
    url := flag.String("url", "", "URL to fetch")
    flag.Parse()
    if *url == "" {
        log.Fatal("url required")
    }
    logger := slog.New(slog.NewTextHandler(os.Stderr, nil))
    fetcher := fetch.NewFetcher(http.DefaultClient, logger)
    data, err := fetcher.Fetch(*url)
    if err != nil {
        logger.Error("fetch failed", "err", err)
        os.Exit(1)
    }
    output.Print(os.Stdout, data)
}
```

No hexagonal ceremony, just practical separation.

**Example 2: Monorepo with a shared library and two services**

```text
platform/
├── libs/
│   └── telemetry/        (module: github.com/co/platform/libs/telemetry)
│       ├── telemetry.go
│       └── go.mod
├── services/
│   ├── auth/
│   │   ├── cmd/auth/
│   │   │   └── main.go
│   │   ├── internal/
│   │   │   ├── service/
│   │   │   └── adapter/
│   │   └── go.mod       (module: github.com/co/platform/services/auth)
│   └── billing/
│       └── ...          (similar)
└── go.work
```

`go.work`:

```go
go 1.22

use (
    ./libs/telemetry
    ./services/auth
    ./services/billing
)
```

Now `main.go` in `auth` can import `github.com/co/platform/libs/telemetry` without a `replace` directive. Each service is independently buildable and can be versioned separately if needed.

**Example 3: Hexagonal service with explicit ports**

```go
// services/auth/internal/core/ports.go
package core

import "context"

type CredentialStore interface {
    Validate(ctx context.Context, user, pass string) (bool, error)
}

// services/auth/internal/core/service.go
package core

type AuthService struct {
    store CredentialStore
}

func NewAuthService(store CredentialStore) *AuthService {
    return &AuthService{store: store}
}

// services/auth/internal/adapters/postgres_store.go
package adapters

import (
    "context"
    "database/sql"
    "github.com/co/platform/services/auth/internal/core"
)

type PostgresCredStore struct{ db *sql.DB }

func (s *PostgresCredStore) Validate(ctx context.Context, user, pass string) (bool, error) {
    // query db...
    return true, nil
}

// Ensure it satisfies the port at compile time.
var _ core.CredentialStore = (*PostgresCredStore)(nil)
```

The core knows nothing about Postgres; the adapter satisfies the interface. In tests, you inject a stub.

## Summary & Exercises

Go’s architecture and project structure revolve around explicit dependencies, compiler-enforced boundaries, and composition. By starting simple, using `internal` to protect your core, and defining interfaces at the consumer, you build systems that are easy to understand, refactor, and scale. The traps are over-engineering (premature layering, excessive packages) and under-engineering (god packages, circular dependencies). The “Go way” is to let the concrete needs of your project drive structure, not a prescriptive blueprint.

**Exercise 1: Refactor a tangled monolith**
You inherit a codebase with a single `main.go` that contains business logic, database access, and HTTP handlers all intertwined, plus a `util` package imported by everything. Design a target structure using `cmd/`, `internal/` with domain packages, and appropriate interfaces. Show a plan for incremental extraction, highlighting how you would break import cycles if they arise. Write the resulting constructor wiring in `main`.

**Exercise 2: Design a service with hexagonal architecture**
Given requirements for a “user notification” service that sends emails, SMS, and push notifications, define the core interfaces (ports) and at least two adapter implementations (one for email via SMTP, one for SMS via an external API). Discuss how you would structure the project to allow swapping implementations without touching business logic, and how you would test the core with mock adapters. Provide a sample test.

**Exercise 3: Monorepo strategy**
Your team is building five microservices that share common logging, tracing, and database migration libraries. Compare the trade-offs of using a single Go module for the whole monorepo versus a separate module per service with `go.work`. Consider binary size, independent deployability, dependency versioning, and developer experience. Outline a `go.work`-based layout and explain when you would move a shared library to its own published module.
