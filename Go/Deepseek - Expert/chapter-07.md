## Chapter 7: Organizing Code with Packages

A package in Go is far more than a directory with a name. It is the fundamental compilation unit, the granularity of dependency management, and the primary mechanism for API boundaries. Every `.go` file begins with a `package` clause, tying it to a specific namespace. The compiler builds one package at a time, resolving imports, verifying types, and generating an object file that the linker later assembles into an executable or shared library. Mastering packages means mastering the structure of any Go program—from a tiny CLI tool to a sprawling monorepo.

---

### Basic Usage

A package declaration binds all `.go` files in the same directory into a single compiled unit. The directory name, by convention, matches the package name. For a library intended for others to import, the package path is its module path joined with the directory inside the module.

```go
// File: $PROJECT/greetings/hello.go
package greetings

import "fmt"

// Hello returns a greeting for the given name.
func Hello(name string) string {
    return fmt.Sprintf("Hello, %s!", name)
}
```

Any other package can import it via the module path:

```go
// File: $PROJECT/cmd/main.go
package main

import (
    "fmt"
    "example.com/myapp/greetings"
)

func main() {
    fmt.Println(greetings.Hello("Gopher"))
}
```

The visibility rule is beautifully simple: an identifier is **exported** if its name starts with an uppercase letter; otherwise, it is **unexported**. There are no `public`, `private`, `protected`, or `friend` keywords.

```go
package shapes

// Exported type
type Rectangle struct {
    // Exported fields
    Width, Height float64
}

// unexported helper
func (r Rectangle) area() float64 {
    return r.Width * r.Height
}

// Exported method
func (r Rectangle) Area() float64 {
    return r.area()
}
```

The `main` package is special: it must contain a `func main()` and produces an executable. Every other package yields a linkable archive.

The `internal` directory further refines visibility. Code residing in a directory named `internal` (at any depth) can only be imported by code rooted at the parent of that `internal` directory. This enforces a true private API boundary within a module.

```
myapp/
    cmd/
        server/main.go
    internal/
        auth/
            tokens.go
    pkg/
        api/
            handlers.go
```

Here, `cmd/server/main.go` can import `myapp/internal/auth`, but any code outside `myapp/` cannot. The `internal` tree is effectively hidden from external consumers.

---

### Under the Hood

When `go build` encounters a package, it constructs a compilation unit: all `.go` files in the directory (excluding `_test.go` files unless testing) are parsed, type-checked, and compiled together into a single object file. The package’s import dependencies form a directed acyclic graph; the compiler walks this graph to ensure each package is built exactly once. Circular dependencies are illegal, caught at compile time with an error like `import cycle not allowed`.

The import path (e.g., `example.com/myapp/greetings`) is resolved to a directory on disk through the module system. The `go.mod` file declares the module path; the `go.sum` file records hashes of dependency contents. When you build, Go checks if the package’s object file in the build cache (`$GOCACHE`) is still fresh by comparing source file hashes, compiler version, and build flags. If so, the cached result is used, dramatically accelerating incremental compilation.

The linker then takes all package archives and the Go runtime, resolves symbols, and produces a static binary (by default). Because each package compiles independently with only imported symbols visible, the compiler can aggressively prune dead code. Unexported functions that are never referenced inside the package, or entire packages that are imported but unused, are eliminated—the famous `imported and not used` error is a compile-time check that also aids dead code elimination.

The `internal` constraint is enforced by the `go` tool, not by the language spec. The tool compares the import path of the consumer against the path of the `internal` directory. The consumer must be in the subtree rooted at the parent of `internal`. This is a simple lexical path prefix check, not a runtime mechanism.

The compiler performs escape analysis within a package boundary, but cross-package inlining requires that the called function’s body be available (via the export data in the object file). The compiler records which functions are inlineable and stores their bodies in the export data. This is why adding a method to an exported type can affect compilation of dependent packages.

---

### Why This Design?

Go’s package model is a deliberate rejection of class-based hierarchies and fine-grained access control. The design rests on three convictions:

1. **Simplicity of visibility**: Exported vs. unexported is binary, trivially understood, and requires no nuanced interpretation. There is no “protected” that leaks implementation details to subclasses, no “package-private” that is confusingly similar to public but subtly different. If something needs to be hidden, it starts with a lowercase letter. The `internal` directory fills the rare need for module-scoped privacy without introducing new keywords.

2. **Packages as the unit of reuse, not classes**: Go has no classes, only types and functions that operate on them. A package groups related functionality around a concept, not a taxonomy. You compose behaviors by embedding types and implementing interfaces, not by extending base classes. This leads to flat, manageable dependency graphs.

3. **Filesystem as namespace**: The import path is a string; there’s no separate “namespace” concept like C++ `namespace` or Python’s module objects. The package name provides a short, local qualifier, while the import path disambiguates packages with the same name. This makes every import self-documenting: `import "net/http"` tells you exactly which package, no aliasing needed unless a collision occurs.

Go’s rejection of a “subpackage” visibility relationship is also intentional. In Java, a protected member is visible to subclasses and to other classes in the same package, which can lead to fragile base class problems. Go avoids this by having no inheritance, and `internal` provides a far simpler, less abuse-prone mechanism.

---

### Competing Approaches

**Java** organizes code with packages and classes. Packages map to directory structures but also serve as access control boundaries: `public`, `protected`, `package-private` (default), and `private`. The `protected` modifier leaks implementation across inheritance hierarchies. Go eliminates inheritance and unifies visibility into two levels, with `internal` serving as an additional compile-time barrier. Java’s package namespace is also typically tied to a reverse domain (e.g., `com.example.myapp`), while Go’s import path is a module path, not necessarily a reverse domain, though best practices encourage unique, domain-based paths.

**Python** uses modules and packages, where a module is a single `.py` file and a package is a directory with `__init__.py`. Everything is public by convention (`_` prefix signals “internal”). Python’s import system is dynamic: `sys.path` can be modified at runtime, making reproducibility a challenge. Go’s module system is fully static and reproducible, with versioning built-in. There’s no runtime import path manipulation; dependencies are resolved at build time.

**C++** has namespaces, which are purely lexical—they disambiguate names but provide no access control. Visibility is managed via `public`, `private`, `protected` on classes. The `friend` keyword allows cross-class access, breaking encapsulation in controlled ways. C++20 modules introduce a proper module system with explicit exports and import, closer to Go’s model, but Go had this from day one with a simpler model.

**Rust** organizes code into modules (files) and crates (packages). Privacy rules are granular: `pub`, `pub(crate)`, `pub(super)`, and private. Rust’s `mod` declarations can be inlined or file-based; the module tree is explicit. Go’s package model is flat by comparison: all files in a directory are peers, no submodule nesting beyond separate directories. Rust’s finer-grained privacy (e.g., crate-visible) resembles Go’s `internal` but is built into the language, not enforced by the tool.

---

### Common Mistakes

**Circular imports** are the most notorious package organization pitfall. Engineers coming from languages that allow mutual references between classes in different files often inadvertently create an import cycle. Since Go compiles whole packages, a cycle between packages A and B (A imports B, B imports A) is forbidden. The solution is to extract a shared interface or type into a third package, or to decouple by using higher-order functions or interface injection.

```go
// Package A
package a
import "example/b"

// Package B
package b
import "example/a" // cycle!
```

**Naming the package after a parent directory** without considering uniqueness. A `util` or `common` package is a code smell because it says nothing about its purpose. Avoid generic package names that will collide with other dependencies or standard library packages (e.g., `errors`, `log`, `testing`). Prefer specific, descriptive names like `textutil` or `ratelimit`.

**Exposing too much API surface** by capitalizing identifiers prematurely. Once exported, a symbol cannot be unexported without a major version bump. Start with unexported names; export only when a real external need exists. This keeps the package’s contract minimal and maintainable.

**Misusing `init()` functions** can lead to hidden side effects and complex initialization ordering. The order of `init()` execution across packages is defined by import order, which can become fragile. Avoid using `init()` for anything other than truly static initialization (e.g., registering drivers for `database/sql`). If you must initialize state, provide an explicit `New()` function that returns a configured instance.

**Confusion between the package name and the directory** is common. The import path (e.g., `example.com/mylib/collections`) refers to the directory; the package name used in code (`collections`) is declared inside the `.go` files. They are conventionally the same, but they can differ. If they do, the code reads awkwardly: `import "example.com/foo"` then `foo.Bar()`, but the import path says `foo`—it’s confusing. Stick to the convention.

**Forgetting that test files with `package foo_test` form an external test package**. This is actually a powerful feature: it forces you to test the exported API only, ensuring your package is usable by outsiders. But many developers accidentally write internal tests in that package and then wonder why they can’t access unexported details.

---

### Performance Considerations

Package design has a direct impact on **compilation speed**. Go’s compiler processes packages in parallel where possible, but a package with a massive number of dependencies forces the compiler to load and type-check many object files. Keeping dependency graphs shallow and packages small reduces incremental build times. There’s no hard limit, but if a package imports 50 others, that’s a signal to refactor.

**Binary size** is affected by what you link. Every exported function, even if never called, is kept if it can be reached from a `main` package through any chain of references (unless stripped by the linker’s dead code elimination, which is conservative). To minimize binary bloat, avoid exposing functions that are only used internally. Use `internal` to hide implementation packages entirely, giving the linker more opportunities to discard them.

**Import cycles** are not a runtime concern—they are prevented entirely. This compile-time enforcement guarantees that dependency graphs are acyclic, which makes parallel compilation and dead code analysis tractable.

**Memory and CPU overhead** at runtime are not directly caused by package structure, but the way you split packages can influence inlining and escape analysis. Functions in different packages cannot be inlined across package boundaries unless their bodies are exported (the compiler records them in the object file). Small, frequently called utilities may benefit from being in the same package to allow cross-function optimization and stack allocation. However, this micro-optimization is rarely the first thing to chase; clear API boundaries should take precedence unless profiling shows a bottleneck.

---

### Best Practices

**Name packages with a single, lowercase word**. `time`, `http`, `json`, `strings`—notice the pattern. No underscores, no camelCase. If a single word isn’t descriptive enough, use a short compound: `textutil`, `ratecounter`. Avoid stealing names from the standard library or well-known community packages.

**Organize by functionality, not by type**. Don’t create a `models` package that holds all structs, a `handlers` package for HTTP handlers, and a `services` package for business logic. That’s Java’s “layer” style. Instead, group by domain: `order`, `user`, `auth`. Each domain package contains its own types, handlers, and logic, exposing only what other domains need. This reduces the number of imports and keeps concerns cohesive.

**Use the `internal` directory to enforce modular boundaries**. If your project has multiple services or commands, place shared implementation details under `internal/...`. For example, a monorepo with multiple binaries might have `internal/database`, `internal/crypto`. Only your own code can import these; external consumers are blocked, discouraging dependency on unstable internals.

**Always document the package** with a `// Package <name> ...` comment placed in any one file (usually a file named `doc.go` or the most central file). `go doc` displays this summary. A well-written package doc explains the purpose and provides a quick example.

**Avoid package-level state**. Global variables in a package (exported or not) make concurrent usage unpredictable and testing painful. Prefer to define types that hold state, and provide constructors. If you must have a singleton, protect it with `sync.Once` and expose it through functions, not direct variable access.

**Keep the `main` package tiny**. The `main` function should parse configuration, construct dependencies (often using explicit dependency injection), and start the application. All real logic belongs in other packages. This ensures reusability and testability.

**Consolidate `.go` files thoughtfully**. A package with 20 files may be fine if they all share a tight theme. But if a single package exceeds a few thousand lines, consider whether it’s doing too much. Splitting into sub-packages can clarify responsibilities, but watch for import cycles.

---

### Examples

**A tiny library with exported API and unexported helpers.**

```go
// Package retry provides a simple retry mechanism.
package retry

import (
    "context"
    "time"
)

// Do calls fn repeatedly until it succeeds or ctx is done.
// It waits a fixed backoff duration between attempts.
func Do(ctx context.Context, backoff time.Duration, fn func() error) error {
    for {
        err := fn()
        if err == nil {
            return nil
        }
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(backoff):
        }
    }
}
```

**Leveraging `internal` for a private rate limiter.**

```
myapp/
    cmd/
        server/main.go
    internal/
        ratelimit/
            limiter.go
    pkg/
        api/
            handler.go
```

`internal/ratelimit/limiter.go`:

```go
// Package ratelimit implements a token bucket rate limiter for internal use.
package ratelimit

import (
    "sync"
    "time"
)

// Limiter controls the rate of operations.
type Limiter struct {
    rate     float64
    burst    int
    tokens   float64
    last     time.Time
    mu       sync.Mutex
}

// NewLimiter creates a Limiter that allows up to burst tokens, refilled at rate per second.
func NewLimiter(rate float64, burst int) *Limiter {
    return &Limiter{
        rate:  rate,
        burst: burst,
        tokens: float64(burst),
        last:  time.Now(),
    }
}

// Allow reports whether an operation may happen now.
func (l *Limiter) Allow() bool {
    l.mu.Lock()
    defer l.mu.Unlock()
    now := time.Now()
    elapsed := now.Sub(l.last).Seconds()
    l.tokens += elapsed * l.rate
    if l.tokens > float64(l.burst) {
        l.tokens = float64(l.burst)
    }
    l.last = now
    if l.tokens >= 1 {
        l.tokens--
        return true
    }
    return false
}
```

The `cmd/server/main.go` can import it freely, but any external project importing `myapp` cannot reach `internal/ratelimit`.

---

### Summary & Exercises

Packages are Go’s unit of code organization, visibility, and compilation. The binary exported/unexported rule, the `internal` directory convention, and the flat, filesystem-mapped structure combine to create a system that is both simple and powerful. A well-designed package tree respects the principles of minimal API surface, domain-driven grouping, and clear dependency flow.

**Exercise 1: Refactor for `internal` boundaries.**
Given a small command-line tool that currently exposes all its components (config parsing, business logic, output formatting) in publicly importable packages, restructure the project to hide everything except the CLI’s entry point and a stable client library. Place configuration parsing, domain models, and formatting helpers under `internal/`. Ensure the `cmd/` package only imports from public packages and `internal/...`. Write a test for the public library that compiles and runs, confirming external consumers cannot access internal code.

**Exercise 2: Eliminate an import cycle.**
Imagine you have two packages: `order` and `user`. `order` needs to notify the user when an order ships, so it imports `user`. But `user` wants to retrieve a user’s orders, so it imports `order`. The cycle prevents compilation. Break the cycle by introducing an interface (e.g., `Notifier`) defined in `order` and implemented by `user`, or by factoring the shared logic into a third package (`orderuser` with interfaces). Implement one solution and verify the build.

**Exercise 3: Design a package with a minimal API.**
Create a package `cache` that provides a thread-safe, in-memory key-value store with TTL expiration. Use only unexported identifiers initially. Then decide which two or three identifiers must be exported to make the package useful: probably a constructor `NewCache`, and methods `Get`/`Set`. Provide a comprehensive package doc comment. Write an external test (in `cache_test` package) that validates the behavior exclusively through the exported API. Refuse to export any internal detail that the test doesn’t need.

By mastering packages, you internalize Go’s philosophy of simple, composable building blocks. Each package becomes a self-contained unit of understanding, and the whole program a structured graph of well-defined dependencies.
