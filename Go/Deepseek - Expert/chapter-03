# Chapter 3: Your First Go Program

Every Go program follows a predictable, almost ritualistic structure. This consistency is intentional: when you’ve seen one Go codebase, you’ve seen the skeleton of them all. Today, we’ll build your first executable, dissect the compilation model, and uncover why Go’s apparent “verbosity” is actually a form of enforced clarity.

---

## 1. Basic Usage

Let’s start with the canonical “Hello, World” – but stripped of magic and annotated for seasoned eyes.

```go
// main.go
package main

import "fmt"

func main() {
    fmt.Println("Hello, World")
}
```

To build and run:

```bash
$ go run main.go          # compile+execute directly (useful for development)
Hello, World

$ go build -o hello main.go   # produce a static binary
$ ./hello
Hello, World
```

### Executables vs. Libraries

The `package main` declaration is special. When the Go compiler sees `package main` with a `func main()`, it produces an **executable** rather than a shared object or static library. Any other package name (e.g., `package math`, `package http`) yields a **library archive** (`.a` file) that can be imported.

**Key nuance:** A package named `main` without a `main()` function compiles but is useless – you can’t link it into anything. The build will succeed, but `go install` will place nothing in `$GOPATH/bin`.

### Multi-file executables

Real programs split across files within the same package:

```go
// greet.go (part of package main)
package main

func greet(name string) string {
    return "Hello, " + name
}
```

```go
// main.go
package main

import "fmt"

func main() {
    fmt.Println(greet("World"))
}
```

Build with `go build .` – the compiler automatically picks up all `.go` files in the current directory (excluding `_test.go` and platform-specific `_windows.go` etc.) that declare `package main`.

---

## 2. Under the Hood

### The Build Process in Detail

Go’s compilation pipeline is refreshingly linear compared to C/C++:

```
Source (.go) → Tokenization → AST → Type Checking → SSA IR → Machine Code → Static Binary
```

**Step 1: Package Resolution**
`go build` reads `go.mod` to determine dependencies, then locates all imported packages (standard library + modules). Unlike C, there’s no separate header step – the compiler parses source files directly, extracting exported symbols from the AST.

**Step 2: Concurrent Compilation**
Go compiles packages **independently** and in parallel, respecting dependency DAG. Each package produces a `.a` (archive) file containing:
- Exported symbols (`//export` directives, public functions/types)
- Debug information (if `-gcflags="-N -l"` disables optimizations)
- Metadata for the linker (relocation records, import list)

**Step 3: Static Linking**
The final link step combines all archives into a single binary. **No dynamic linking by default** (except for `//go:cgo_import_dynamic` or `-buildmode=c-shared`). This includes the Go runtime (scheduler, GC, stack management), the entire standard library used, and even `fmt.Printf`’s parsing tables.

**Result:** A standalone binary with zero runtime dependencies – copy it to a `scratch` Docker container or an ancient Linux server, and it runs unchanged.

### The `go run` Illusion

`go run` doesn’t interpret the source. It:
1. Compiles the program to a temporary directory (e.g., `/tmp/go-build123/b001/exe/main`)
2. Executes it
3. Removes the binary on exit (unless `-work` flag is passed)

This makes `go run` unsuitable for production – you lose the binary. Use it only for development scripts or quick experiments.

### Why No Build Scripts (Make/CMake)?

Go’s toolchain infers everything from the source. The compiler determines dependencies by parsing `import` statements – no header files, no `-I` paths, no `-l` linker flags for standard libraries. The only common flag is `-ldflags "-X main.version=1.0"` to inject values at link time.

**Aha moment:** Go treats your entire workspace as a database of packages. The toolchain is the query engine.

---

## 3. Why This Design?

### Why `package main` and not a `main` function in any package?

Most languages (C, C++, Java, C#) use a naming convention – `main` or `Main` – in a global scope. Go requires an explicit **package main** for two reasons:

1. **Explicit build target:** When you see `package main`, you immediately know this directory produces an executable, not a library. This is a compile-time contract.
2. **No link-time surprises:** In C, you can compile multiple `main()` functions into separate object files, then link one. Go prevents this – the `main` package can appear only once in a build graph.

### Why no dynamic linking by default?

Simplicity over flexibility. Dynamic linking introduces:
- Dependency version hell (DLL hell)
- Platform-specific loading mechanics (`LD_LIBRARY_PATH`, `PATH`, `rpath`)
- Performance penalties (PLT indirections, lazy binding)

Go’s answer: **build everything once, ship a fat binary**. This mirrors how Google deploys services – containers with static binaries, not shared libraries. The trade-off is binary size (~2–3 MB for a minimal HTTP server, ~10–15 MB for typical CLI tools) and duplication of the runtime across processes (though kernel-level page sharing mitigates this for identical binaries).

### Why does every Go program look similar?

Go enforces a **canonical file structure**:
- No `#define` or macros – use constants and functions
- No header files – `export` is implicit via capital letters
- No multiple inheritance or constructor hierarchies – just `init()` functions (one per file, executed in dependency order)

This consistency means you can navigate any Go project immediately. The `go fmt` tool eliminates stylistic debates – the community agreed to one formatting rule.

---

## 4. Competing Approaches

| Language | Build Model | Entry Point | Dependency Management |
|----------|-------------|-------------|----------------------|
| **Go** | Static binary, single-step (`go build`) | `func main()` in `package main` | `go.mod` (MVS) |
| **C/C++** | Separate compile+link, headers | `int main(int argc, char** argv)` | Make/CMake + system libs |
| **Rust** | Static by default, but supports `cdylib` | `fn main()` | `Cargo.toml` |
| **Java** | JARs / classpath, JIT-compiled | `public static void main(String[] args)` | Maven/Gradle (transitive) |
| **Python** | Interpreted, no explicit entry | `if __name__ == "__main__":` | `pip` + virtualenvs |

### Java vs. Go: The JAR vs. Binary

Java’s model prioritizes **bytecode portability** (run anywhere with a JVM). Go prioritizes **deployment simplicity** – no JVM to patch, no classpath debugging, no `NoClassDefFoundError`. However, Go binaries are OS- and architecture-specific: `GOOS=linux GOARCH=amd64 go build` yields a binary that won’t run on ARM or Windows.

### C vs. Go: Header Files

C separates declaration (`.h`) from implementation (`.c`). Go avoids this duplication – the compiler extracts exported symbols directly from source. This eliminates:
- Include guard errors
- Mismatched declarations
- Forward declarations for circular dependencies (Go handles circular imports with a compile-time error, forcing better design)

**Trade-off:** Go’s approach requires parsing all source files even for small changes, but incremental compilation (Go 1.10+) caches package archives, making rebuilds fast.

### Python vs. Go: Entry Point Clarity

Python’s `if __name__ == "__main__":` pattern is flexible – the same file can be a module or a script. Go’s separation is rigid: `package main` is **never** importable by other packages. This prevents accidental misuse but requires a separate `cmd/` directory if you want both a library and an executable from the same codebase.

---

## 5. Common Mistakes

### Mistake 1: Multiple `main()` functions in a package

```go
// main.go
package main
func main() { println("first") }

// other.go
package main
func main() { println("second") }
```

**Error:** `main redeclared in this block`. Go requires exactly one `main()` per package.

### Mistake 2: Using `init()` when a simple constructor would do

```go
var db *sql.DB

func init() {
    db, _ = sql.Open("sqlite3", "file:db.sqlite") // error ignored!
}
```

`init()` runs before `main()`, but:
- Order across files is **deterministic but not portable** (lexical file name order)
- Error handling is awkward – you can’t return errors
- Initialization happens even if your binary has subcommands that don’t need the DB

**Better:** Use a constructor function called explicitly from `main()`.

### Mistake 3: Not understanding `go run`’s temporary nature

Newcomers often write scripts with `go run` and assume the binary persists. Then in CI/CD, they compile repeatedly, wasting time. **Rule:** Use `go run` only for local development; in Dockerfiles or build pipelines, always `go build`.

### Mistake 4: Assuming `package main` can be imported

```go
// mylib.go (in package main)
package main

func Helper() {} // Exported (capital H)
```

```go
// otherpackage.go
import "github.com/user/myproject" // imports package main?
```

**Result:** Compilation fails with `import "..." is a program, not an importable package`. The `main` package is a leaf in the dependency graph – nothing can depend on it.

---

## 6. Performance Considerations

### Compilation Speed

Go’s compiler is famously fast. Let’s measure:

```bash
$ time go build -o /dev/null ./hello
real    0m0.123s
```

Compare with Rust (debug build):
```bash
$ time cargo build
real    0m1.892s   # cold, with dependencies
```

Why is Go faster?
- **No header parsing:** Source files are the only input.
- **Simplified grammar:** Fewer language features reduce AST complexity.
- **Parallel compilation:** Each package builds independently.
- **SSA backend:** Fast lowering to machine code without heavy optimization passes (unless `-O2` is requested, which Go defaults to – but optimization is still cheaper than C++’s templates).

### Binary Size

Minimal “Hello World” sizes:

| Language | Stripped Binary Size (Linux x86_64) |
|----------|--------------------------------------|
| C (`gcc -Os`) | ~16 KB |
| Rust (`--release`) | ~400 KB |
| **Go** (`-ldflags="-s -w"`) | ~1.2 MB |
| C++ (`g++ -static`) | ~800 KB |

Go’s larger size includes:
- Runtime scheduler (~200 KB)
- GC infrastructure (~150 KB)
- Stack management and panic handling
- `fmt` package’s reflection tables (despite only using `Println`)

**Mitigation:** Use `-ldflags="-s -w"` to strip debug symbols and DWARF tables. For extreme cases, `tinygo` (LLVM-based) produces sub-100 KB binaries but lacks full standard library support.

### Start-up Latency

Go binaries start almost instantly because:
- No dynamic linker overhead
- No JIT warmup (Java’s C2 compiler)
- Initial heap is tiny (configurable via `GOGC`)

Benchmark:

```bash
$ time ./hello
real    0m0.002s (2 ms)
```

Python’s interpreter startup alone is ~50 ms; a Spring Boot app takes 1–3 seconds.

---

## 7. Best Practices

### 1. Use `package main` only once per executable project

Organize larger CLIs with subcommands:

```
cmd/
  myapp/
    main.go          # package main
  myappctl/
    main.go          # separate executable
internal/
  shared/
    lib.go           # package shared (not main)
```

### 2. Keep `main()` minimal

`main()` should parse flags, initialize logging, and call into a `run()` function that returns an error:

```go
func main() {
    if err := run(); err != nil {
        slog.Error("fatal", "error", err)
        os.Exit(1)
    }
}

func run() error {
    // business logic starts here
    return nil
}
```

This enables testing `run()` from a `_test.go` file – you cannot directly call `main()` in tests.

### 3. Use `go mod init` before writing code

Even for a “Hello World”:

```bash
$ go mod init hello
```

This enables module-aware builds and pins your dependencies. Without it, `go build` still works but uses legacy `GOPATH` mode (deprecated in Go 1.16+). For Go 1.21+, modules are mandatory.

### 4. Leverage `//go:embed` for static assets (Go 1.16+)

```go
package main

import _ "embed"

//go:embed greeting.txt
var greeting string

func main() {
    print(greeting)
}
```

This embeds files into the binary – no external asset deployment.

### 5. Always commit `go.sum`

`go.sum` contains cryptographic hashes of each dependency version. It protects against:
- Man-in-the-middle attacks on module proxies
- Accidental version changes

Never `rm go.sum` – treat it like a lockfile.

---

## 8. Examples

### Example 1: Command-line argument parsing

```go
package main

import (
    "flag"
    "fmt"
    "log"
    "os"
)

func main() {
    // Define flags
    name := flag.String("name", "World", "person to greet")
    repeat := flag.Int("repeat", 1, "number of times to greet")
    flag.Parse()

    // Validate
    if *repeat < 1 {
        log.Fatal("repeat must be >= 1")
    }

    for i := 0; i < *repeat; i++ {
        fmt.Printf("Hello, %s!\n", *name)
    }
}
```

Build and test:
```bash
$ go build -o greet
$ ./greet -name Alice -repeat 3
Hello, Alice!
Hello, Alice!
Hello, Alice!
```

### Example 2: Multi-file package with `init()` (use sparingly)

```go
// config.go
package main

import "os"

var config struct {
    host string
    port string
}

func init() {
    config.host = os.Getenv("APP_HOST")
    if config.host == "" {
        config.host = "localhost"
    }
    config.port = os.Getenv("APP_PORT")
    if config.port == "" {
        config.port = "8080"
    }
}
```

```go
// main.go
package main

import "fmt"

func main() {
    fmt.Printf("Running on %s:%s\n", config.host, config.port)
}
```

**Critique:** `init()` makes testing harder – the environment is read before `TestMain` can set it. Prefer explicit initialization in `main()`.

### Example 3: Building for multiple platforms

```bash
# Linux amd64
GOOS=linux GOARCH=amd64 go build -o myapp-linux

# Windows amd64
GOOS=windows GOARCH=amd64 go build -o myapp.exe

# macOS ARM (M1/M2)
GOOS=darwin GOARCH=arm64 go build -o myapp-macos
```

The same source compiles everywhere – no `#ifdef` needed thanks to Go’s platform abstraction (`os`, `syscall` packages).

---

## 9. Summary & Exercises

### Summary

- **Every Go executable** is defined by `package main` containing `func main()`.
- **Static linking** is the default, producing single-binary deployments.
- **Compilation speed** comes from concurrent package compilation and no header files.
- **Consistency** across Go programs (`go fmt`, canonical structure) reduces cognitive load.
- **The entry point** is minimal – real logic lives in a `run()` function returning an error.

### Key Takeaways for Seasoned Engineers

- Go’s build model trades binary size for deployment simplicity – a deliberate choice for server-side software.
- The absence of preprocessor macros and dynamic linking eliminates an entire class of configuration bugs.
- `go run` is not a REPL or interpreter – it’s a compile-execute wrapper with temporary binaries.

### Exercises

#### Exercise 1: Build a `wc`-like tool

Write a Go program that reads from `os.Stdin` (or a file named on the command line) and prints line, word, and byte counts (like `wc`). Use only the standard library. Requirements:
- Support `-l` (lines only), `-w` (words only), `-c` (bytes only) flags (default to all three).
- Use `bufio.Scanner` for lines and `strings.Fields` for words.
- Implement error handling: if the file doesn’t exist, print the error to `stderr` and exit with code 1.

**Hint:** The `flag` package can define multiple flags; test with `go run main.go -- -l README.md` (the `--` stops flag parsing).

#### Exercise 2: Understand init order

Create three files in the same `package main`:
- `a.go` with `func init() { println("a") }`
- `b.go` with `func init() { println("b") }`
- `main.go` with `main()`

Run `go build` multiple times, rename files, and observe the order of `init()` execution. Then read the spec: “init functions are executed in the order they appear in the source”. What does “appear” mean – lexical file name order, or order passed to the compiler? Verify with `go build -work` and examine the temporary directory’s file list.

#### Exercise 3: Cross-compile and inspect

Build your solution from Exercise 1 for Windows (even on macOS/Linux):
```bash
GOOS=windows GOARCH=amd64 go build -o wc.exe
```

Run `file wc.exe` (on Unix) or examine it with `hexdump -C | head`. Then build a Linux version and compare sizes with `ls -lh`. Why is the Windows binary slightly larger? (Hint: Look up “PE vs. ELF alignment”.)

---

**Next Chapter Preview:** *Variables, Constants, and Types* – where we’ll dissect Go’s type inference, zero values, and why `var x int` is not the same as `x := 0` under the hood.
