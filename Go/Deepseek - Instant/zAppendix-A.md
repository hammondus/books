## Appendix A: Go Tooling Deep Dive (`go build`, `go vet`, `go work`)

The Go toolchain is not an afterthought—it is a deliberate part of the language’s design. Unlike ecosystems where build systems, linters, and dependency managers are bolted on by the community (e.g., Python’s `pip` + `venv` + `pytest` + `black`), Go ships a unified, zero‑friction set of tools that work the same way on every machine.

This appendix goes deep into three critical tools:
- **`go build`** – the compilation driver that turns packages into binaries or libraries.
- **`go vet`** – the static analyzer that catches correctness bugs.
- **`go work`** – the multi‑module workspace manager (Go 1.18+).

We will also touch on supporting commands (`go list`, `go mod graph`, `go tool compile`) to understand what happens beneath the surface.

---

### A.1 `go build` – The Compilation Engine

#### Basic Usage

Build the current package into an executable (default name = last component of the import path):
```bash
go build
```

Build a specific package:
```bash
go build ./cmd/server
```

Cross‑compile for a different OS/architecture (without installing a separate toolchain):
```bash
GOOS=linux GOARCH=arm64 go build -o server-linux-arm64 ./cmd/server
```

Build a shared library (`.so` / `.dylib`) instead of an executable:
```bash
go build -buildmode=c-shared -o libmymath.so ./mymath
```

#### Under the Hood: The Build Cache & Compilation Steps

`go build` is a clever caching system layered on top of the underlying compiler (`compile`) and linker (`link`). The process:

1. **Load the package graph** – `go list` resolves imports recursively.
2. **Compute the build hash** – includes source files, compiler flags, environment, and transitive dependencies’ hashes.
3. **Check `$GOCACHE`** – if the hash exists, reuse the already compiled object file (`.a` archive).
4. **Compile each package in parallel** – `compile` writes object files into the cache.
5. **Link** – `link` combines the main package’s object file with runtime and dependencies into a final binary.

The cache is deterministic and content‑addressable. You can inspect it:
```bash
go env GOCACHE          # usually ~/.cache/go-build
go clean -cache        # delete all cached objects
go clean -testcache     # only test results
```

**Key internals:**
- **Export data** – each package archive contains a compressed description of its public API. The compiler uses this for incremental builds without re‑parsing dependent code.
- **Compilation flags** – `-N` disables optimisations (useful for debugging), `-l` disables inlining, `-m` prints optimisation decisions (escape analysis, inlining).
- **Linking** – Go binaries are statically linked by default, but you can opt into dynamic linking with `-buildmode=pie` (position‑independent executable) or cgo dependencies.

#### Why This Design?

The Go team prioritised **reproducible builds** and **developer speed**. By embedding caching and deterministic hashing into the toolchain, they eliminated the need for external build systems like Make or CMake.
> *“Why not rely on the host system’s build tools?”*
> Because cross‑platform consistency matters more than following Unix traditions. A Go build on Windows produces the same binary as on Linux (given the same GOOS/GOARCH).

#### Competing Approaches

| Language | Build Model | Cache? | Cross‑compile? |
|----------|-------------|--------|----------------|
| **Go**   | Unified tool `go build` | Yes, automatic | First‑class (`GOOS`/`GOARCH`) |
| **Rust** | `cargo build`, separate `rustc` | Yes, incremental | Via `--target` (requires target std libs) |
| **C/C++**| Make/CMake + compiler driver | Manual (ccache) | Heavy (need sysroots) |
| **Java** | `javac` + `jar`, build tools (Maven/Gradle) | Per‑tool caching | Separate JDK per platform |

Rust comes closest, but its caching requires explicit configuration of `target/` directories. Go’s cache is global and transparent.

#### Common Mistakes

- **Forgetting `-o`** – building a package with `go build ./pkg` produces an executable named `pkg` (on Unix) or `pkg.exe` (on Windows), even if `pkg` is a library. Use `go build -o /dev/null ./pkg` to just cache the package without a binary.
- **Build tags inconsistency** – build tags (e.g., `//go:build linux`) change the set of compiled files. If you forget to set `-tags=` consistently, you may get subtly different binaries.
- **`go build` with relative imports** – using `go build -o app .` inside `cmd/app` is fine, but `go build -o app ../..` breaks because the import path is ambiguous. Always reference packages by absolute import path or use `./...`.

#### Performance Considerations

- **Cache hits** are free. On a warm cache, `go build` on a medium‑sized project (100k LOC) takes <0.5s.
- **Link time** dominates for large binaries (e.g., 100MB+). Use `-ldflags="-s -w"` to strip debug symbols and reduce binary size (up to 30%).
- **Parallel compilation** – GOMAXPROCS controls compiler concurrency. For CI runners, set `GOMAXPROCS` to the number of available cores.
- **`go build -a`** forces recompilation of all packages, ignoring the cache. Only useful when you suspect cache corruption or when messing with compiler internal flags.

#### Best Practices

- **Always version your binaries** – inject version info via `-ldflags`:
  ```bash
  VERSION=$(git describe --tags)
  go build -ldflags="-X main.version=$VERSION" ./cmd/myapp
  ```
- **Use `-trimpath`** – removes absolute file system paths from the binary, improving reproducibility and reducing binary size.
- **For production binaries**, combine `-ldflags="-s -w"` and `-trimpath`:
  ```bash
  go build -ldflags="-s -w" -trimpath -o myapp ./cmd/myapp
  ```
- **Store `go build` commands in a Makefile** only if you need complex flags – otherwise a `build.sh` script or `//go:generate` directive is more idiomatic.

#### Examples

**Example A1: Building a multi‑binary monorepo**
```go
// cmd/api/main.go
package main

func main() { println("api") }

// cmd/worker/main.go
package main

func main() { println("worker") }
```
```bash
# Build both binaries in one go
go build -o bin/api ./cmd/api
go build -o bin/worker ./cmd/worker
```

**Example A2: Conditional compilation with build tags**
```go
// file_darwin.go
//go:build darwin

package main

func init() { println("macOS specific setup") }
```
```bash
# Build for macOS (automatically picks up the tag)
go build -o myapp .
```

---

### A.2 `go vet` – The Sanity Checker

#### Basic Usage

`go vet` examines Go source code and reports suspicious constructs that are not compilation errors but likely bugs.

```bash
go vet ./...           # vet all packages in the module
go vet -tags=integration ./pkg/...
go vet -v ./...        # verbose: list each package as it is vetted
```

#### Under the Hood: How `vet` Works

`vet` operates on the compiler’s **abstract syntax tree (AST)** and **type information** (the same `go/types` package used by IDEs). It runs a set of analysers, each implementing a single check.

List all available analysers (Go 1.21+):
```bash
go tool vet help
```

Notable analysers:
- `printf` – checks format strings against arguments (e.g., `fmt.Printf("%s", 123)`).
- `shadow` – detects variable shadowing (`-shadow` flag, but now part of `govet` in 1.22+? Actually `shadow` is separate in `golang.org/x/tools/go/analysis/passes/shadow`, not enabled by default).
- `unreachable` – unreachable code after `panic` or `return`.
- `loopclosure` – captures loop variables incorrectly in goroutines or `defer`.
- `nilfunc` – calling a function that is always nil (rare).
- `tests` – checks for common test mistakes (e.g., `(*T).Error` without `Errorf`).

When you run `go vet`, it executes all default analysers. You can enable extra ones:
```bash
go vet -vettool=$(which shadow) ./...
```

#### Why This Design?

Go’s philosophy: **linting should be in the core, not a hundred community tools**. `go vet` catches *real bugs* (not style issues) without configuration. Style is handled by `go fmt`. This reduces decision fatigue: one command for correctness, one for formatting.

> *“Why not just rely on the compiler?”*
> Compilation ensures type safety, but not semantic correctness. `printf` misuse is legal but wrong. `go vet` fills the gap without requiring a separate static analysis tool.

#### Competing Approaches

| Language | Built‑in Bug Checker | Style Enforcement |
|----------|----------------------|--------------------|
| **Go**   | `go vet` | `go fmt` |
| **Rust** | `cargo check` (compiler warnings) + `clippy` | `rustfmt` (separate) |
| **Python** | None – rely on `pylint`, `mypy`, `flake8` | `black` |
| **Java** | `javac` limited warnings; `SpotBugs` / `ErrorProne` separate | `google-java-format` |

Rust’s `clippy` is more extensive (hundreds of lints) but requires a separate installation. Go’s `vet` stays minimal by design – only the checks the Go team considers universally beneficial.

#### Common Mistakes

- **Assuming `vet` catches all bugs** – It does not catch deadlocks, race conditions (use `go run -race`), or logic errors.
- **Running `vet` only on changed files** – `./...` is cheap and safe, so always run on the whole module.
- **Ignoring `vet` warnings in CI** – Make `go vet` a hard failure. Warnings indicate real problems that will bite you in production.

#### Performance Considerations

- `go vet` is **fast** – it shares the same parse cache as `go build`. On a warm cache, it finishes in <0.1s per package.
- For large codebases (>500k LOC), vetting all packages sequentially is still I/O bound; use `go vet -n` to see the commands and run them in parallel with `xargs` if needed (rare).
- The `shadow` analyser is slower because it builds additional scoping information; enable it only when necessary.

#### Best Practices

- **Run `go vet` in your pre‑commit hook and CI pipeline** before `go test`.
- **Use `go vet` with `//go:build ignore`** – you can vet test files by including `./...` – it automatically includes `_test.go` files.
- **Add `vet` to your editor** – VSCode with the Go extension runs `go vet` on save.
- **Create custom analysers** – the `golang.org/x/tools/go/analysis` framework lets you write your own vet checks. Integrate them with:
  ```bash
  go vet -vettool=$(which myvet) ./...
  ```

#### Examples

**Example A3: `vet` catches a classic loop closure bug**
```go
func main() {
    for i := 0; i < 3; i++ {
        go func() {
            fmt.Println(i)   // i is captured by reference, all print 3
        }()
    }
    time.Sleep(1 * time.Second)
}
```
```bash
$ go vet loopclosure.go
# go vet reports:
# ./loopclosure.go:5:12: loop variable i captured by func literal
```

**Example A4: Misused printf**
```go
func log(msg string) {
    fmt.Printf(msg, "extra")   // vet: missing argument for Printf format
}
```
```bash
$ go vet printf.go
# ./printf.go:3:2: fmt.Printf format %s reads arg #1, but call has 1 arg
```

---

### A.3 `go work` – Multi‑Module Workspaces

#### Basic Usage

`go work` (introduced in Go 1.18) allows you to work on multiple modules simultaneously without publishing `replace` directives to `go.mod`.

Create a workspace:
```bash
go work init ./module-a ./module-b
```
This generates a `go.work` file:
```go
go 1.21

use (
    ./module-a
    ./module-b
)
```

Add another module later:
```bash
go work use ./module-c
```

Sync dependencies across workspace modules:
```bash
go work sync
```

#### Under the Hood: How Workspaces Override the Module Graph

When a `go.work` file is present in the current directory or any parent, the `go` command activates **workspace mode**. The workspace file tells the toolchain to treat the listed local directories as *replacement roots* for their respective modules.

During dependency resolution:
1. The `go` command reads `go.work` and creates an **overlay** of modules.
2. Any import path that matches a `use` directive resolves to the local directory (instead of the version from the module proxy).
3. The `go.sum` of each module is still respected, but the workspace ignores checksum mismatches for replaced modules (a feature, not a bug).

Crucially, `go work` **does not modify** any `go.mod` file. This keeps your published module clean.

#### Why This Design?

Before workspaces, multi‑module development required `replace` directives in `go.mod`:
```go
replace example.com/foo v1.2.3 => ../foo
```
This polluted version control and broke CI (the `../foo` path may not exist on the build server).
`go work` moves that local development overrides **out of the module file** and into a **developer‑only** workspace file (typically ignored by `.gitignore`).

> *“Why not just use a monorepo?”*
> Go modules encourage per‑repo versioning, but many organisations (Google included) operate huge monorepos. `go work` gives monorepo teams the same ergonomics as a single‑repo developer.

#### Competing Approaches

| Ecosystem | Multi‑module local dev |
|-----------|-------------------------|
| **Go**    | `go work` |
| **Rust**  | `[patch]` section in `.cargo/config.toml` or `path` dependencies |
| **Node.js** | `npm link` or `yarn workspaces` |
| **Python** | `pip install -e ../other-module` or `poetry` path dependencies |

Rust’s `path` dependencies are closest, but they still require editing `Cargo.toml`. Go’s `go work` is non‑invasive and lives outside version control.

#### Common Mistakes

- **Committing `go.work` to version control** – Unless you are building a team‑shared monorepo, `go.work` should be in `.gitignore`. Each developer’s local paths differ.
- **Forgetting to `go work sync`** – When you change dependencies in one workspace module, other modules still see the old version until you run `sync`.
- **Conflicting with `replace` directives** – If a module in the workspace has a `replace` directive that points to another module also in the workspace, the behaviour is undefined. Keep `replace` for external overrides only.

#### Performance Considerations

- Workspace mode adds negligible overhead – the `go` command simply resolves local directories.
- However, `go list` may show different dependency versions than a normal build. Always run `go test` inside the workspace mode.
- For CI, avoid workspaces – use the published module versions. Workspaces are for **local development only**.

#### Best Practices

- **Use `go work` for multi‑module projects during development** (e.g., a service that depends on an in‑house library).
- **Add `go.work` to `.gitignore`** unless you standardise the absolute paths across your team (e.g., everyone clones repos into `~/dev/`).
- **Use `go work vendoring`** – you can vendor dependencies inside a workspace, but it’s rarely needed.
- **Clean up unused workspace entries** with `go work edit -dropuse=./old-module`.

#### Examples

**Example A5: Developing a library and a service simultaneously**
```
~/dev/
  mylib/
    go.mod (module github.com/me/mylib)
  myservice/
    go.mod (module github.com/me/myservice)
```
```bash
cd ~/dev
go work init ./mylib ./myservice
cd myservice
go run main.go   # uses local ./mylib instead of the remote version
```

**Example A6: Workspace with a replaced standard library?** (advanced)
You can even replace `std` with a local checkout of Go source:
```go
go 1.21
use ./myservice
replace std => /go/src   # only works with `-tags=goexperiment`
```
*(This is for Go compiler developers, not normal users.)*

---

### A.4 Supporting Tooling – Quick Reference

| Command | Purpose | Most useful flag |
|---------|---------|------------------|
| `go clean` | Remove object files & cache | `-modcache` (purge module cache) |
| `go env` | Print Go environment variables | `GOOS`, `GOARCH`, `GOPROXY` |
| `go generate` | Run generators (e.g., `stringer`) | `-run` regex to select directives |
| `go get` | Add / upgrade dependencies (Go 1.21+ prefers `go get` for tools only) | `tool` (for binaries) |
| `go install` | Compile and install executables to `GOBIN` | `@latest` version |
| `go list` | List packages, their dependencies, and metadata | `-json` for machine consumption |
| `go mod tidy` | Trim `go.mod` and add missing requirements | `-diff` shows changes |
| `go test` | Run tests and benchmarks | `-cover`, `-race`, `-bench=.` |

---

### Summary & Exercises

#### Key Takeaways
- **`go build`** caches aggressively; use `-ldflags="-s -w" -trimpath` for production binaries.
- **`go vet`** catches semantic bugs; run it in CI alongside `go test`.
- **`go work`** enables painless multi‑module development without touching `go.mod`.

#### Exercises

**Exercise A1: Build optimisation challenge**
Take a non‑trivial Go project (e.g., `github.com/hashicorp/consul`). Measure its build time with `time go build .`. Then apply `-ldflags="-s -w"` and `-trimpath`. How much does the binary size shrink? How much does build time change? (Hint: linking is the expensive part – stripping debug info reduces link time marginally.)

**Exercise A2: Write a custom `go vet` analyser**
Using `golang.org/x/tools/go/analysis`, write an analyser that flags any function named `Temp` (a common abbreviation for “temporary”) that does not have a comment explaining its purpose. Use the `run` function to inspect the AST and print a diagnostic. Install it as a separate binary and run it with `go vet -vettool=./myvet ./...`.

**Exercise A3: Simulate a workspace upgrade workflow**
Create two modules: `logger v1.0.0` (provides `Log(string)`) and `app` that imports `logger`. Upgrade `logger` to v2.0.0 with a breaking change (`Log(ctx context.Context, msg string)`). Use `go work` to point `app` to the local `logger` v2 source. Without changing `app/go.mod`, modify `app`’s code to use the new API. Prove that `go run` inside the workspace succeeds while `go build` outside (with the published v1) fails.
