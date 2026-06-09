## 3. Your First Go Program

Go treats the developer experience as a first-class concern, and nothing exemplifies this better than the path from a single source file to a running, statically linked binary. This chapter unpacks the simplest possible Go program—not to teach you what a `print` statement is, but to reveal the toolchain, compilation model, and philosophy baked into every executable you’ll ever write.

---

### Basic Usage

A Go executable lives in a file declared `package main` and must contain a function `func main()`. The canonical “Hello, World” looks like this:

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, World!")
}
```

To run it without producing a persistent binary:

```
go run main.go
```

To compile and keep the artifact:

```
go build -o hello main.go
./hello
```

`go build` produces a statically linked native binary by default. You can drop that binary on any machine with the same OS/architecture and it will run—no interpreter, no virtual machine, no shared libraries (unless you explicitly link against C libraries via CGO). The `go` command is the single entry point to the entire workflow: formatting, testing, building, and dependency management all live under the same CLI.

The file name is arbitrary; `go build` looks at the package declaration, not the file name. Conventions like `main.go` are just conventions. You can spread the `main` package across multiple `.go` files inside the same directory if you wish, as long as exactly one file provides `func main()` (and none of the files violate the “one package per directory” rule).

---

### Under the Hood

When you invoke `go build`, several stages fire in sequence:

1. **Source parsing & type checking:** The compiler reads every `.go` file in the package, resolves imports, and builds an abstract syntax tree. Type checking happens early—unused imports or variables cause a compilation *error*, not a warning.

2. **Compilation to object code:** Each package compiles to an archive (`.a` file) containing the package’s compiled code and export data. The standard library is already pre-compiled in the Go installation cache, so `fmt` doesn’t need to be rebuilt.

3. **Linking:** The linker pulls together the `main` package, all transitive dependencies, and the Go runtime. The runtime includes the garbage collector, goroutine scheduler, and bootstrap code. The final binary is statically linked by default on Linux and macOS; even on platforms that default to dynamic linking, Go prefers static linking to simplify deployment.

At program startup the execution sequence is:

- The kernel loads the binary and jumps to `_rt0_<arch>_<os>` (the platform-specific entry point written in assembly).
- The runtime initializes: sets up thread-local storage, spawns the initial OS thread, initializes the garbage collector, creates the scheduler, and sets up signal handling.
- `runtime.main` is called on the main goroutine. It runs `main.init()` functions (package-level initialization) and finally invokes `main.main()`.

This explains why a simple “Hello, World” binary is roughly 1.2 MB (Go 1.21, linux/amd64, stripped): the runtime is always included. There is no way to compile a “runtime-less” Go executable. The trade-off is that every Go binary carries a rich set of concurrent primitives, a precise GC, and a scheduler, even if you only print a single line.

---

### Why This Design?

Newcomers often ask: “Why do I have to write `package main` and `func main()`? Why not just a top-level script like Python?” The answer is **explicitness**, which is a central Go value.

- **Unambiguous entry point:** There is no `if __name__ == "__main__"` magic. The compiler enforces that exactly one `main` function exists across all files in package `main`. This eliminates edge cases where a file can accidentally behave differently when executed directly versus imported.
- **Packages as the universal unit of code:** Every Go file belongs to a package. Even a one-liner script is a package. This consistency means the same tooling (testing, vetting, documentation) applies uniformly—no separate “script mode.”
- **Import paths are strings:** `import "fmt"` looks like a file path, but it’s a logical package import path resolved by the compiler and the module system. There are no classpath tricks, no alias-by-default, and no wildcard imports. You always see exactly where symbols come from.
- **All Go programs look the same:** `gofmt` enforces a single canonical style. Combined with mandatory package clauses and import grouping, Go codebases—whether from Google, a startup, or an open-source project—share a visual rhythm. This reduces the time needed to become productive in an unfamiliar codebase.

This design philosophy stems from Google’s need to maintain massive, multi-decade codebases where engineers frequently switch teams. A language that mandates uniformity at the syntactic level directly addresses the organizational scaling problem.

---

### Competing Approaches

| Language | Minimal “Hello, World” | Notable differences |
|----------|------------------------|----------------------|
| **Go**   | `package main; import "fmt"; func main() { fmt.Println("Hello") }` | Statically linked binary, no runtime dependency. Compiler enforces style. |
| **Python** | `print("Hello")` | Terse, but requires an interpreter and often a virtual environment. Execution semantics change if the file is imported. |
| **Java**  | `public class Main { public static void main(String[] args) { System.out.println("Hello"); } }` | Requires a class and `String[] args`, JVM, compilation to bytecode, and classpath management. |
| **C**     | `#include <stdio.h>; int main(void) { printf("Hello\n"); return 0; }` | Tiny binary (~16 KB), but must manage headers, linking, and platform differences manually. |
| **Rust**  | `fn main() { println!("Hello"); }` | Similar minimal syntax, but `println!` is a macro; builds via Cargo; binaries carry no GC but have larger debug info. |
| **Node.js** | `console.log("Hello");` | Requires a JavaScript runtime (V8). No compilation step, but startup time and memory footprint are higher. |

Go deliberately rejects the “script” model: there is no interpreter that skips compilation. Even `go run` compiles to a temporary binary and executes it. The single binary output, with zero external dependencies, makes Go especially suited for containers, CLI tools, and microservices where shipping a whole runtime (like the JVM or Python’s `site-packages`) is undesirable.

---

### Common Mistakes

- **Forgetting `package main` or misnaming it:** If the package declaration is `package myapp`, `go build` will produce no error (it’s a valid library), but running `go run` will complain: `package is not a main package`. Seasoned engineers switching from languages without package-level entry points often trip here.
- **Omitting `func main`:** The compiler error `missing function main` is clear, but the fix may not be obvious if `main` is defined in another file that isn’t compiled (e.g., a file with a build tag that doesn’t match).
- **Unused imports and variables:** Go refuses to compile code with unused imports or variables. After commenting out a `fmt.Println` call, you must also remove `"fmt"` from the import block. The `goimports` tool automates this, but the strictness surprises engineers from permissive ecosystems.
- **Shadowing the package name:** Declaring a variable named `fmt` (e.g., `fmt := "text"`) will shadow the import and cause `fmt.Println` to fail. The compiler error will say `fmt.Println undefined (type string has no field or method Println)`, which can be cryptic at first glance.
- **Attempting to run a library:** Running `go run` on a file with `package mylib` produces `package is not a main package`. This reinforces that executables and libraries are fundamentally distinct.
- **Filename issues:** Files ending in `_test.go` are excluded from regular builds. Files with platform suffixes like `_linux.go` or `_windows.go` are conditionally compiled based on `GOOS`. Putting `main` in a file that doesn’t match the target platform will cause a “missing function main” error.

---

### Performance Considerations

**Compilation time:** Go’s compiler is designed for speed. Incremental builds are near-instant for small projects because the compiler caches package outputs aggressively. Even a full build of a Hello World from clean cache takes roughly 100 ms on modern hardware.

**Binary size:** A stripped Hello World binary on linux/amd64 is about 1.2 MB. This includes the runtime, GC, scheduler, and all types used by `fmt`. While larger than a C equivalent, it is dwarfed by the disk and memory footprint of a JVM or Node.js runtime. For CLI tools where size matters, you can reduce it further with `-ldflags="-s -w"` (strip debug information) and by using `print`/`println` built-ins instead of `fmt` (though that sacrifices functionality). UPX compression can bring the binary under 500 KB, but at a startup latency cost.

**Startup time:** The runtime initialization is highly optimized and typically completes in tens to hundreds of microseconds. The dominant cost for a trivial program is dynamic linker time (on platforms that use it) and page faults when loading the binary. In containerized environments, Go’s static linking often eliminates the dynamic linker step entirely. For performance-sensitive CLI tools (e.g., `kubectl`, `docker`), Go’s startup is fast enough that it rarely becomes a bottleneck—unlike JVM-based CLIs, which routinely incur hundreds of milliseconds of warm-up.

**Memory footprint:** The “Hello, World” process consumes roughly 0.5–1 MB of resident memory at steady state. The runtime pre-allocates a small heap and a handful of goroutine stacks. This overhead is constant and amortized over the lifetime of long-running services.

---

### Best Practices

- **Keep `main` minimal:** The `main` function should parse flags, initialize logging, and call into a library function. Business logic should never live in the `main` package. This makes the core logic testable and reusable.
- **Use the `cmd` directory convention:** In multi-binary projects, place each executable’s `main` package under `cmd/<binary-name>/main.go`. This keeps the repository root clean and signals which directories produce artifacts.
- **Leverage `goimports`:** Automatically add and remove imports on save. This eliminates friction from Go’s strict unused-import rule.
- **Favor `_` for side-effect imports sparingly:** The blank identifier import (e.g., `import _ "net/http/pprof"`) registers handlers or drivers. Use it only when the side effect is necessary and document it explicitly.
- **Use `flag` or environment variables for configuration:** Avoid hardcoding values. The standard `flag` package is sufficient for most CLIs; for complex applications, consider `os.Getenv` with sensible defaults.
- **Let the toolchain manage formatting:** Run `gofmt` or `go fmt` before committing. Formatting is not a matter of taste in Go—it’s automated and non-negotiable. This consistency is a feature.

---

### Examples

A more realistic first program that reads a name from a flag or environment variable, timestamps the output, and handles basic errors:

```go
package main

import (
	"flag"
	"fmt"
	"os"
	"time"
)

func main() {
	// Default to the USER environment variable; override with -name flag.
	name := flag.String("name", os.Getenv("USER"), "name to greet")
	flag.Parse()

	if *name == "" {
		fmt.Fprintln(os.Stderr, "name must not be empty")
		os.Exit(1)
	}

	fmt.Printf("[%s] Hello, %s!\n", time.Now().Format(time.RFC3339), *name)
}
```

Run it:

```
$ go run greet.go -name=Gopher
[2026-06-09T14:22:05Z] Hello, Gopher!
```

This snippet demonstrates:

- Multiple imports grouped in a factored import block.
- The standard `flag` package for flag parsing.
- Access to environment variables with `os.Getenv`.
- Writing to stderr and exiting with a non-zero status on failure.
- Using `time.Now().Format` with the `time.RFC3339` constant.

All of this uses only the standard library. The binary produced is completely self-contained and can be distributed without worrying about runtime compatibility.

---

### Summary & Exercises

**Recap:**

- Every Go executable begins with `package main` and `func main()`. The toolchain (`go run`, `go build`) enforces this contract.
- The build process produces a statically linked binary that embeds the Go runtime, giving you a single deployable artifact with no external dependencies.
- Go’s uniformity—enforced by `gofmt`, the package system, and import rules—ensures that every program you encounter feels familiar, lowering the cognitive cost of code navigation.
- While the binary is larger than a C equivalent, it’s still tiny compared to shipping a full language runtime, and the startup latency is negligible for nearly all use cases.

**Exercises:**

1. **Environment printer:** Write a program that reads all environment variables via `os.Environ()`, sorts them lexicographically, and prints each on its own line in `KEY=VALUE` format. Time the startup of the resulting binary with `time ./envprint`. How does it compare to `env` from coreutils?

2. **Mini-CLI with subcommands:** Using only the `os.Args` slice (no external framework), create a program that accepts a subcommand: `time` (prints the current time in RFC3339), `date` (prints the date only), and `greet <name>` (prints a greeting). If no subcommand is given, print a usage message. Consider how you would test the different subcommand paths.

3. **Compile an HTTP server from scratch:** Write a `main` function that starts an HTTP server on `:8080` and responds with `"Hello, World!"` to every request (use `net/http`). Build the binary, run it, and hit it with `curl`. Then stop and reflect: this binary contains a production-grade HTTP stack, a scheduler, and a GC—all in a few lines of code and one self-contained file. How would you replicate this deployment simplicity in Java? In Python?
