## Chapter 7: Organizing Code with Packages

Packages are Go’s only mechanism for code structuring, encapsulation, and reuse. Unlike languages that offer classes, namespaces, modules, and packages as distinct concepts, Go collapses all of these needs into one unified abstraction: the package. This chapter explains how to use packages effectively, why Go’s design deliberately rejects richer hierarchical constructs, and how to avoid common pain points when coming from class‑based or namespace‑heavy ecosystems.

### 1. Basic Usage

A package is a collection of `.go` files stored in the same directory. Every file begins with a `package` clause, followed by imports, then declarations.

**Declaring a package**

```go
// file: arithmetic/add.go
package arithmetic

// Add is exported (capitalized)
func Add(a, b int) int {
    return a + b
}

// multiply is unexported (lowercase) – visible only inside package arithmetic
func multiply(a, b int) int {
    return a * b
}
```

**Importing and using a package**

```go
// file: main.go
package main

import (
    "fmt"
    "yourmodule/arithmetic" // import path = module path + subdirectory
)

func main() {
    sum := arithmetic.Add(5, 3)
    fmt.Println(sum) // 8
    // arithmetic.multiply(5,3) // compile error: cannot refer to unexported name
}
```

**The `internal` package**

Directories named `internal` are special: their contents can only be imported by code inside the parent of that `internal` directory.

```
yourmodule/
  internal/
    auth/       // only importable by code inside yourmodule/
      token.go
  public/
    api.go      // can import "yourmodule/internal/auth"
  main.go       // can also import "yourmodule/internal/auth"
```

Any code outside `yourmodule/` (e.g., `othermodule/`) that tries to import `yourmodule/internal/auth` will be rejected by the compiler.

**Package‑level initialization** – `init()` functions run automatically, once per package, after all variable declarations have been evaluated.

```go
package cache

var globalCache map[string]interface{}

func init() {
    // Guaranteed to run before any code in this package is called.
    globalCache = make(map[string]interface{}, 100)
}
```

### 2. Under the Hood

**Import path resolution.** When the compiler sees `import "yourmodule/arithmetic"`, it translates that string into a filesystem path relative to the module root (or `GOPATH` in legacy mode). The import path never dictates the package’s API—only the `package` clause inside the files determines the package name used by the importer. This decoupling allows renaming a package without changing its import path (via `import arithmetic_v2 "yourmodule/arithmetic"`).

**Compilation unit.** All `.go` files belonging to the same package are compiled together as one unit. Identifiers (types, functions, variables) declared in any of those files are shared across all of them, without any forward‑declaration requirement. This is why you can split a large package into several files arbitrarily.

**Visibility enforcement.** The rule is purely lexical: an identifier starting with an uppercase Unicode letter is exported; lower‑case is unexported. The compiler checks visibility at compile time based on the package path of the referencing code. There is no runtime access control or reflection‑based bypass (except via `unsafe` or `reflect` with tricks – but that is considered hostile).

**`init()` ordering.** When a package is imported, Go computes a dependency graph of all imported packages, then initializes them in a depth‑first order. Inside a single package, `init()` functions run in the order of source file names (lexicographically), and within one file, in the order of declaration. You cannot rely on inter‑file ordering; if you need deterministic sequencing, combine initialisation into a single `init()` or, better, into an explicit `Setup()` function called from `main()`.

**The `internal` mechanism.** The compiler hard‑codes the rule: an import of a path containing `/internal/` is allowed only if the importing package’s path is a prefix of the parent of that `/internal/`. This is enforced during import resolution, not at runtime. It is a syntactic guarantee, not an obfuscation – tools can still see the code, but the compiler refuses to link it.

### 3. Why This Design?

**Why no classes?** Classes bundle state and behaviour into a single inheritance‑prone unit. Go packages instead group related functionality, and types (structs) inside a package do not have privileged access to each other beyond normal visibility rules. This forces explicit composition and keeps dependency graphs flat. The team believed that classes encourage deep, brittle hierarchies, whereas packages encourage reuse through clean, decoupled APIs.

**Why no namespaces?** Languages like C++ and C# have namespace hierarchies (`std::vector`, `System.Collections.Generic`). Go deliberately rejects hierarchical namespaces because they create an illusion of organisation that often breaks down in practice (e.g., `utils.net`, `utils.db`, `utils.string` – all growing without discipline). Instead, Go forces all package names to be short, meaningful, and local. The import path provides the hierarchical *location*, but the package name in code remains flat. This reduces cognitive load: you never guess whether the current file is in a sub‑namespace.

**Why explicit exporting?** Java uses `public`/`private`/`protected` modifiers; Python uses a leading `_` convention that is not enforced. Go makes exporting a single, unambiguous rule: case of the first letter. This eliminates the need for keywords like `export` or visibility modifiers, and makes it immediately obvious when reading a declaration whether it is part of the public API.

**Why no circular imports?** Go rejects circular imports at compile time. This forces a clean dependency graph. In languages that allow cycles (Java, Python), you often end with subtle initialisation order bugs or “partially constructed” objects. Go’s simplicity here eliminates an entire class of runtime failures.

**The `internal` convention as a design tool.** `internal` was added after years of experience with large monorepos. It gives library authors a way to hide implementation details from external users while still sharing them across different parts of their own project – something that pure visibility (unexported) cannot do because unexported identifiers are only visible within a single package. With `internal`, you can break your code into many packages but still maintain a clean public API boundary.

### 4. Competing Approaches

| Language | Unit of Organisation | Visibility Control | Circular Deps | Initialisation Order |
|----------|----------------------|--------------------|---------------|----------------------|
| **Go** | Package (directory) | Case‑based export | Forbidden | Deterministic, depth‑first |
| **Java** | Package + Class | `public`/`private`/`protected` | Allowed (but warned) | Class‑by‑class, can be subtle |
| **Python** | Module + Package (`__init__.py`) | Convention (`_` prefix) + name mangling | Allowed (runtime import) | Module‑level code runs at import |
| **C++** | Namespace + Translation unit | `public`/`private` in classes; `export` rarely used | Allowed (via headers) | Static initialisation order fiasco |
| **Rust** | Crate + Module | `pub` keyword | Allowed (but careful) | Fixed order, but with lazy statics |

**Java vs. Go.** Java packages are primarily a namespace to avoid name collisions, but the real encapsulation boundary is the class (and its `private` fields). Go elevates the package to the primary boundary. Java’s `protected` allows access to subclasses, which often leads to tight coupling; Go has no inheritance, so this problem never arises.

**Python vs. Go.** Python’s runtime imports and dynamic nature mean that cyclic imports often fail only when the program is executed. Go catches cycles at compile time. Python relies on naming conventions (`_func`) for “private” APIs; Go enforces visibility with compiler errors.

**C++ vs. Go.** C++ has the “static initialisation order fiasco” across translation units. Go avoids this by guaranteeing that all `init()` functions run after all imported packages have been fully initialised, and by disallowing any implicit ordering dependencies within a package.

**Rust vs. Go.** Rust’s module system is more expressive (you can re‑export names, control visibility with `pub(crate)`, etc.). Go deliberately trades expressiveness for predictability: the mapping from import path to filesystem and the simple export rule mean there is only one way to organise code.

### 5. Common Mistakes

**Stuttering names.** Because the package name is already part of the qualified identifier, repeating it in the type or function name leads to stuttering.

```go
// bad
package log
type Logger struct { ... }
func NewLogger() *Logger { ... }

// good
package log
type Logger struct { ... }
func New() *Logger { ... }
```

The caller writes `log.New()` not `log.NewLogger()`. This is a strong convention in Effective Go.

**Using relative imports.** `import "./arithmetic"` works but is brittle and deprecated. Always use full module‑relative paths. Relative imports break when you move files or switch to a different build system.

**Overusing `init()`.** Newcomers put complex initialisation (reading config, opening DB connections) inside `init()`. This makes testing impossible and hides dependencies. Reserve `init()` for trivial, panic‑free setup of package state (e.g., regexp compilation, flag parsing after `flag.Parse()` is not safe in `init` – use `main()`).

**Circular imports.** Suppose `package A` imports `package B`, and `package B` imports `package A`. The compiler rejects this even if the cycle is indirect (A→C→B→A). The fix is to extract the shared types or functions into a third package `common` or restructure the design – often an indication that the two packages should be merged.

**Misunderstanding `internal` scope.** Developers sometimes place `internal` at the root of a module and then wonder why other packages inside the same module cannot import it – they can, as long as the importer’s path is a prefix of the parent of `/internal/`. With module‑root `internal/`, *any* package inside the module can import it. To restrict visibility to a subtree, place `internal` deeper.

**Package name collisions with stdlib.** Naming your package `http` or `json` is allowed but leads to confusing code (e.g., `http.MyClient` vs `net/http`). Use more specific names: `myhttp`, `apiclient`, etc.

### 6. Performance Considerations

**Compile time impact.** Go’s compiler processes each package independently and caches results in `$GOPATH/pkg` (or module cache). Splitting a large package into many small packages increases the number of compilation units, which can increase parallelisation but also adds overhead for many tiny imports. As a rule of thumb, keep packages logically cohesive – dozens of packages are fine; thousands may slow down incremental builds.

**Inlining and cross‑package optimisation.** The Go compiler (since 1.20) can inline small functions across package boundaries if they are marked with `//go:noinline` or if the heuristics decide it’s profitable. However, unexported functions are easier to inline because the compiler sees all call sites within the package. Exporting a function solely for inlining across packages is rarely worth it – rely on the compiler’s mid‑stack inlining (enabled by default).

**Linker overhead.** Every exported symbol adds an entry to the symbol table. While not a concern for typical applications, thousands of exported functions from hundreds of packages can increase binary size and link time. The linker’s dead code elimination (`-ldflags=-w -s`) strips debug info but cannot remove all unused exported symbols because they might be accessed via reflection.

**`init()` cost.** Each `init()` function runs sequentially and cannot be parallelised. Many packages with non‑trivial `init()` can noticeably increase program startup time. Measure with `go test -bench=. -benchtime=1s` if startup latency matters.

### 7. Best Practices (Idiomatic Go)

**One package per directory.** Never put multiple `package` declarations in the same directory. The directory name should match the package name, except for `package main` (which can reside in a directory named `cmd/myapp`).

**Keep package names short and lowercase.** Use `http`, `json`, `fmt` – not `HTTP`, `JsonParser`, `FormattedIO`. Acronyms should be all upper or all lower: `url` not `URL` (except when exported: `URL` is fine). Avoid underscores and mixedCaps.

**Design for zero‑valued usability.** A package should export types that can be used without complicated constructors.

```go
package counter

type Counter struct {
    mu sync.Mutex
    n  int
}

func (c *Counter) Inc() {
    c.mu.Lock()
    c.n++
    c.mu.Unlock()
}
```

Caller can simply write `var c counter.Counter` – the mutex is zero‑valued and ready to use.

**Avoid `util`, `common`, `base` packages.** These become dumping grounds for unrelated code and create hidden dependency cycles. Instead, move the utility function to the package that owns the domain (e.g., `strings.Repeat` lives in `strings`, not `util`). If a function is truly generic, consider whether it belongs in a new, well‑named package.

**Return concrete types, accept interfaces.** Even within a package, it’s often cleaner to return structs (so methods can be added later) and accept interfaces only when you genuinely need abstraction.

**Place `main()` packages in `cmd/`.** The common pattern:

```
yourmodule/
  cmd/
    myapp/
      main.go   // package main
    mytool/
      main.go   // package main
  pkg/
    api/        // reusable library
    internal/   // private to yourmodule
```

This makes it easy to build multiple binaries from the same repository.

**Use `go list` to inspect your package graph.** `go list -f '{{.ImportPath}} {{.Imports}}' all` shows dependencies. `go mod why -m <module>` explains why a dependency is needed.

### 8. Examples

**Example 1: A simple `config` package with internal helpers**

```go
// internal/fileutil/read.go
package fileutil

import "os"

func ReadFile(path string) ([]byte, error) {
    return os.ReadFile(path)
}
```

```go
// config/loader.go
package config

import (
    "encoding/json"
    "yourmodule/internal/fileutil"
)

type Config struct {
    Port int    `json:"port"`
    Name string `json:"name"`
}

func Load(path string) (*Config, error) {
    data, err := fileutil.ReadFile(path)
    if err != nil {
        return nil, err
    }
    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        return nil, err
    }
    return &cfg, nil
}
```

**Example 2: Avoiding circular imports – refactoring shared types**

Before (illegal):

```go
// a/a.go
package a
import "yourmodule/b"
func A() { b.B() }
```

```go
// b/b.go
package b
import "yourmodule/a"
func B() { a.A() }
```

After:

```go
// shared/types.go
package shared
type Work struct{ ID int }
```

```go
// a/a.go
package a
import "yourmodule/shared"
func A(w shared.Work) { /* ... */ }
```

```go
// b/b.go
package b
import "yourmodule/shared"
func B(w shared.Work) { /* ... */ }
```

**Example 3: Using `internal` for a database driver**

```
db/
  internal/
    conn/
      pool.go     // unexported connection management
    wire/
      protocol.go // unexported wire format
  mysql.go        // exports public API
  postgres.go
```

Code outside `db/` cannot import `db/internal/conn`, guaranteeing encapsulation.

### 9. Summary & Exercises

**Summary**

- Packages are the sole organisation unit in Go: one directory = one package.
- Exported identifiers start with an uppercase letter; unexported with lowercase.
- Import paths are strings; the package name inside the files determines how callers refer to it.
- `internal` directories enforce visibility boundaries across packages within the same module.
- Circular imports are forbidden – a feature, not a limitation.
- Keep package names short, avoid stuttering, and prefer concrete returns over interfaces.

**Exercises**

1. **Detect and break a cycle.** Write two packages, `order` and `customer`, where `order` needs `customer.ID` and `customer` needs `order.Total` to compute a loyalty discount. The compiler rejects the cycle. Refactor into a third package `types` that holds shared data structures, and move business logic into separate packages that both depend on `types`. Verify with `go build`.

2. **Build a thread‑safe cache with internal eviction.** Create a package `cache` that exports a `Cache` type with `Set(key, value)` and `Get(key)` methods. Internally, use a separate `internal/evict` package that implements an LRU policy. The `cache` package should import `evict`, but external users of `cache` must not be able to call `evict.LRU`. Write a test that demonstrates the public API works and that the `evict` package cannot be imported by a `main` package outside `cache/`.

3. **Refactor a Java‑style utility package.** Given a legacy Go codebase with a package `util` containing `StringToInt`, `ReadJSONFile`, `RetryHTTP`, and `MD5Hash`, split it into three well‑named packages (`strconv`, `ioext`, `retry`, `crypto`). Update all callers. Measure the compilation time before and after – the difference should be negligible, but the code becomes self‑documenting.
