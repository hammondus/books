# Chapter 3: Your First Zig Program

Every systems language makes a statement in its entry point. C says: "you receive raw arguments and return a number; anything else is convention." Rust says: "failure here is a panic unless you explicitly opt into `Result`." Go says: "there is nothing to return; communicate failure through the process." Zig says: "you may fail, and if you do, that is a value — not a panic, not a convention, not undefined behavior."

That statement is encoded in five characters: `!void`. Before you write your first meaningful program, it is worth understanding what those characters mean, why they were chosen, and what they reveal about the language beneath them.

---

## 1. Basic Usage

### The Entry Point

The minimal compilable Zig program is:

```zig
const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    try stdout.print("Hello, world!\n", .{});
}
```

Save this as `hello.zig` and run it directly:

```
$ zig run hello.zig
Hello, world!
```

Or compile and run separately:

```
$ zig build-exe hello.zig
$ ./hello
Hello, world!
```

Every element here is deliberate. Dissecting top to bottom:

**`const std = @import("std")`** imports the Zig standard library. `@import` is a built-in function — not a preprocessor directive — that returns the imported file as a **namespace: a compile-time struct type**. The name `std` is a convention, not a keyword; you could write `const zig_stdlib = @import("std")` and use `zig_stdlib.io` throughout. The point is that the standard library is not special. It is a collection of Zig files, and you access it exactly as you would access any module you write yourself.

**`pub fn main() !void`** is the entry point contract. Each word carries weight:

- `pub` exports the symbol so the linker can find it. Without `pub`, `main` is private to its file and the linker will fail to locate an entry point.
- `fn main()` is a zero-argument function. Command-line arguments are not passed as parameters; they are retrieved explicitly from the process environment when you need them.
- `!void` is the return type. The `!` prefix denotes an **error union** — this function either returns `void` (normal completion) or an error value. This is not special entry-point syntax; it is the same `!T` error union type that works identically everywhere in Zig.

**`std.io.getStdOut().writer()`** obtains the standard output file handle and wraps it in a generic writer interface. This is explicit: there is no global `print` function. You get a handle and write to it.

**`try stdout.print("Hello, world!\n", .{})`** formats and writes the string. `try` propagates any I/O error up to `main`. The `.{}` is an empty anonymous struct, which is the correct argument for a format string with no format specifiers.

### The `std.debug.print` Shorthand

For quick diagnostics, `std.debug.print` is available without ceremony:

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("Hello from stderr\n", .{});
}
```

This writes to **stderr**, not stdout. It acquires a mutex internally to provide thread safety. For production output, use `std.io.getStdOut().writer()`; use `std.debug.print` only for development diagnostics and temporary logging. The distinction matters in pipelines: anything written to stderr does not appear in `$(command substitution)` or through pipe chaining.

Note that this version of `main` returns `void` because `std.debug.print` panics on write failure rather than returning an error. Whether that trade-off is acceptable depends on your reliability requirements — for a debug print in development code, it typically is.

### Reading from Standard Input

Reading from stdin follows the same explicit pattern:

```zig
const std = @import("std");

pub fn main() !void {
    const stdin = std.io.getStdIn().reader();
    var buf: [256]u8 = undefined;
    const line = try stdin.readUntilDelimiterOrEof(&buf, '\n');
    if (line) |l| {
        const stdout = std.io.getStdOut().writer();
        try stdout.print("You said: {s}\n", .{l});
    }
}
```

`readUntilDelimiterOrEof` returns `!?[]u8` — an error union wrapping an optional slice. `try` handles the error case; the `if (line) |l|` syntax unwraps the optional. If EOF is reached before a newline, `line` is `null` and the block is skipped cleanly.

The `{s}` format specifier tells the formatter to interpret the argument as a UTF-8 string — a byte slice. Without it, the formatter would attempt to print each byte individually as a character, which is almost never what you want.

### Command-Line Arguments

Zig does not pass `argc`/`argv` to `main`. You retrieve arguments explicitly:

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer {
        const check = gpa.deinit();
        if (check == .leak) @panic("memory leak detected");
    }
    const allocator = gpa.allocator();

    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);

    const stdout = std.io.getStdOut().writer();
    for (args, 0..) |arg, i| {
        try stdout.print("args[{d}] = {s}\n", .{ i, arg });
    }
}
```

`std.process.argsAlloc` allocates a slice of null-terminated strings, one per argument. The zeroth argument is the program name. `argsFree` releases the allocation. Memory allocation is explicit, paired, and visible in the source.

`for (args, 0..) |arg, i|` iterates the slice with an index counter. The `0..` syntax produces an implicit range starting at zero, zipped with the slice. This is the idiomatic Zig pattern for indexed iteration — no separate counter variable, no `enumerate` adapter.

### Initializing a Project with `zig init`

For anything beyond a single throwaway file, use `zig init` to scaffold the standard project layout:

```
$ mkdir my-project && cd my-project
$ zig init
```

This generates:

```
my-project/
├── build.zig          # Build script
├── build.zig.zon      # Package manifest
└── src/
    ├── main.zig       # Executable entry point
    └── root.zig       # Library root (for mixed exe+lib projects)
```

The generated `build.zig` defines build targets in executable Zig code. Running `zig build run` from the project root compiles and executes `src/main.zig`. The build system is covered in depth in Chapter 30; for now, the important point is that `zig init` gives you a working project with a correct `build.zig` that you can modify immediately.

---

## 2. Under the Hood

### The Compilation Pipeline

When you run `zig run hello.zig`, the compiler executes a multi-stage pipeline. Understanding each stage helps you reason about compile-time errors, debug information, incremental compilation, and performance characteristics of the resulting binary.

**Stage 1: Tokenization**

The source file is scanned into a flat token stream. Zig's token set is small and regular — there is no preprocessor, no macro expansion, and no conditional compilation at this layer. Every token corresponds directly to text in the source file. The tokenizer is deterministic and produces the same output regardless of the surrounding context.

**Stage 2: Parsing → AST**

Tokens are parsed into an **Abstract Syntax Tree (AST)**. Zig's parser is entirely deterministic and does not require type information to produce the AST. The parser can run to completion on any syntactically valid source file before any type information is known. This is what makes `zig fmt` fast and reliable: formatting requires only parsing, not semantic analysis or type checking.

The AST is stored in a compact, arena-allocated format. Function bodies are encoded as sequences of flat integer indices into node arrays rather than heap-allocated linked trees. The layout is cache-friendly and fast to traverse — a design decision that pays dividends during repeated analysis in large codebases.

**Stage 3: AstGen → ZIR**

The AST is lowered to **ZIR (Zig Intermediate Representation)**, an untyped, instruction-based IR. ZIR captures syntactic structure and control flow without resolving types or evaluating comptime expressions. This is the representation that `zig ast-check` validates for surface-level structural correctness without performing a full compilation.

ZIR is also the format that the build cache stores for incremental compilation. If a source file's ZIR has not changed since the last build — even if the file was reformatted, changing whitespace or comment text — downstream re-analysis is skipped. This is the foundation of Zig's incremental compilation: changes that do not affect the semantic content of a file do not trigger recompilation of its dependents.

**Stage 4: Sema → AIR**

Semantic analysis transforms ZIR into **AIR (Analyzed Intermediate Representation)**. This stage performs the bulk of the compiler's work:

- Resolves all type references and performs type checking
- Evaluates all `comptime` expressions, blocks, and parameters
- Specializes generic functions for their concrete type arguments
- Performs control flow analysis, eliminating comptime-dead branches
- Inserts runtime safety checks for `Debug` and `ReleaseSafe` builds: bounds checks on slice accesses, integer overflow detection, null pointer checks, unreachable-branch panics

AIR is fully typed: every instruction has a concrete, known type. The compiler knows the size, alignment, and ABI representation of every value in the program at this stage. There are no `void*`-like escape hatches in AIR; the type information is complete.

**Stage 5: Codegen → Machine Code**

AIR is lowered to machine code through one of two backend paths:

- **Self-hosted backends** (available for x86_64, aarch64, arm, RISC-V, WASM, and others in Zig 0.14): These produce object files directly from AIR, without going through LLVM. They are fast — debug compilation is dramatically faster than the LLVM path — but produce less optimized code. The self-hosted x86_64 backend can compile non-trivial programs in tens of milliseconds.
- **LLVM backend**: AIR is translated to LLVM IR, which LLVM then optimizes and lowers to machine code. This path is used for `ReleaseFast`, `ReleaseSafe`, and `ReleaseSmall` builds, where optimization quality matters more than compilation speed.

In Zig 0.14, `Debug` builds default to the self-hosted backend on supported targets, which dramatically reduces compile-to-run latency for development cycles. Release builds unconditionally use LLVM.

**Stage 6: Linking**

The compiled object files are passed to the linker. Zig includes its own linker implementation (and wraps platform linkers where needed) that resolves symbol references between compilation units, applies relocations, and produces the final executable in the platform-native format: ELF on Linux, Mach-O on macOS, PE/COFF on Windows.

On Linux, the default output is a dynamically linked ELF binary against glibc. For fully static binaries with no external runtime dependencies, pass `-target x86_64-linux-musl` to link against musl libc statically. For bare-metal targets, no libc is linked at all.

### The Build Cache

Zig's global cache at `~/.cache/zig` (or `%LOCALAPPDATA%\zig\zig-cache` on Windows) uses **content-addressable storage**. Each cached artifact is identified by a hash of its inputs: source content, compiler version, build flags, and dependency hashes. This gives the cache several useful properties:

- Switching between debug and release builds does not invalidate the other mode's cached artifacts.
- Running `zig build` from two different branches of a project will share artifacts that happen to be identical — if both branches have the same version of a dependency, that dependency is compiled once.
- The cache is safe to delete at any time. `rm -rf ~/.cache/zig` costs you a full rebuild but nothing more: all information needed to reconstruct the artifacts is in the source tree.

### The Entry Point Bootstrap

When you write `pub fn main() !void`, you are not writing the machine-code entry point. That is `_start` on ELF platforms (or `mainCRTStartup` on Windows), provided by the Zig standard library in `lib/std/start.zig`. The bootstrap layer:

1. Sets up the initial stack frame and stack guard page
2. Retrieves `argc`/`argv`/`envp` from the OS-provided startup state
3. Calls platform initialization routines
4. Calls your `main()` and catches its return value

If `main` returns an error, the bootstrap writes the error name to stderr — for example, `error.OutOfMemory` or `error.FileNotFound` — and calls `std.process.exit(1)`.

If `main` returns normally, the bootstrap calls `std.process.exit(0)`.

This is entirely visible source code. You can open `lib/std/start.zig` in the Zig installation directory and read exactly what happens between `_start` and your `main`. The standard library is not a black-box runtime. It is Zig code, and in extreme cases — embedded targets, custom OS ports — you can replace it entirely by providing your own `_start` and controlling the bootstrap process.

### How `@import` Works

`@import("std")` is resolved at compile time. The compiler locates the standard library path relative to the Zig installation, parses and semantically analyzes `lib/std/std.zig`, and returns a reference to the resulting type. The import is cached: regardless of how many files in your project write `@import("std")`, the library is analyzed once per compilation.

`@import("./utils.zig")` works analogously for local files. The path is resolved relative to the file containing the `@import`. Relative paths (`./` prefix) and bare names (`"utils.zig"`) are both supported; the former is conventional for clarity.

The critical insight: every imported file is, in Zig's type system, **a `struct` with `pub` declarations as its fields**. This is not a metaphor or an approximation. `@import("std")` returns a type. `std.io` is a field access on that type. `std.io.getStdOut` is a further field access. You can pass `@import` results to comptime functions, store them in comptime variables, and inspect their declarations with `@typeInfo` — the same tools that work on any struct work on modules, because they are the same thing.

### What the Binary Contains

A `zig build-exe hello.zig` in `Debug` mode produces a binary that includes:

- Your compiled code
- Only the standard library functions transitively called from `main` — Zig performs dead code elimination by default
- Full DWARF (Linux/macOS) or CodeView (Windows) debug information
- Runtime safety check code: bounds checks, overflow checks, unreachable-branch panics

Representative binary sizes for hello-world on Linux x86_64:

| Build mode | Linkage | Approximate size |
|---|---|---|
| `Debug` | dynamic (glibc) | ~250 KB |
| `Debug` | static (musl) | ~700 KB |
| `ReleaseFast` | static (musl) | ~8 KB |
| `ReleaseFast` + `-fstrip` | static (musl) | ~4 KB |

The `Debug` binary is large because it contains full debug information, and the safety check infrastructure adds code at every potentially-unsafe operation. The `ReleaseFast` binary is small because all of that is stripped, dead code is eliminated, and LLVM optimization reduces the program to its essential machine instructions.

---

## 3. Why This Design?

### Why `pub fn main() !void` Instead of `int main()`?

C's `int main(void)` predates standardized error handling. The return value is an exit code — a number from 0 to 255 — which is a convention, not a type-system guarantee. Nothing prevents returning 42 on success and 0 on failure; the compiler will not object. More critically, functions called inside `main` that fail silently do not propagate their failure to the process exit code in any enforced way. The discipline of checking every return value is maintained by the programmer, not the language.

Zig's `!void` makes the compiler enforce acknowledgment of every error. If a function returns `!void` and you call it with `try`, any error it produces becomes an error at the call site. The chain from leaf function to `main` is type-checked end to end. The exit-code logic lives in the bootstrap, not in application code, and it is consistent across all programs.

Returning `void` instead of an integer removes a source of confusion: the exit code is not your function's return value. If you need to control the exit code explicitly, call `std.process.exit(n)` — the intent is obvious in the source.

### Why No Arguments to `main`?

C passes `argc` and `argv` to `main` because that is the most direct path from the kernel's process creation ABI. It is also the only thing you get: environment variables require a separate call to `getenv` or access to the non-standard `envp` third parameter.

Zig puts all process information retrieval behind explicit calls: `std.process.argsAlloc`, `std.process.getEnvMap`, `std.process.getEnvVarOwned`. This is consistent with the principle that **implicit inputs to a function should not exist**. The process environment is an input; requiring you to fetch it explicitly makes that dependency visible in the source. You can read a Zig function's signature and know exactly what it receives — there are no ambient, invisible inputs.

### Why `@import` Instead of `#include`?

`#include` in C performs **textual substitution**. The preprocessor copies the contents of the header file verbatim into the translation unit. This has well-known compounding costs:

- Every translation unit that includes a large header re-parses and re-analyzes it, even when the header has not changed
- Include order affects correctness — include guards (`#ifndef GUARD_H`) exist to compensate for a mechanism that inherently allows multiple inclusion
- Everything in a header is visible to everything that includes it; there is no controlled visibility

`@import` in Zig is a **semantic import**. The imported file is parsed and analyzed once per compilation. The result is cached. Circular imports are detected at compile time and rejected. `pub` controls visibility — declarations not marked `pub` in an imported file are inaccessible to importers, enforced by the compiler.

The practical consequence: large Zig codebases do not suffer from the "millions of lines parsed per translation unit" problem that plagues large C++ projects. Headers like `<windows.h>` or `<boost/...>` add hundreds of milliseconds to compilation per file that includes them. Zig has no equivalent pathology.

### Why Is Every File a Struct?

Making every file a struct is an elegant unification: the module system uses the same mechanism as ordinary data structures. `@import("foo.zig")` and `const MyType = struct { ... }` are both types. You can pass them to comptime functions, store them in comptime variables, and iterate their declarations with `@typeInfo(@TypeOf(@import("foo.zig")))`.

This means there is no privileged "module" concept distinct from the type system. The tools that work on types work on modules, because they are the same abstraction. There is no separate module introspection API to learn; if you understand Zig's comptime type reflection (covered in Chapter 7), you already understand module reflection.

### Why No Global `print`?

Languages like Python and Go provide top-level `print`/`fmt.Println` that "just works." This convenience hides details:

- **Where does the output go?** You must read documentation to know it is stdout.
- **Can it fail?** In Go, `fmt.Println` silently discards write errors. In Python, it can raise `IOError`, which you may or may not catch.
- **Is it buffered?** That depends on the runtime and platform.

Zig forces all of these questions into the source code. `std.io.getStdOut().writer()` tells you exactly where you are writing. `try stdout.print(...)` forces you to acknowledge the possibility of failure. If you want buffering, you wrap in `std.io.bufferedWriter`. Nothing is hidden, and the reader of the code can reason about I/O behavior without consulting runtime documentation.

---

## 4. Competing Approaches

### C

```c
#include <stdio.h>

int main(int argc, char *argv[]) {
    printf("Hello, world!\n");
    return 0;
}
```

C's entry point is explicit about arguments but implicit about errors. `printf` can fail — writing to a closed pipe or a full disk returns a negative value — but that return is almost universally ignored. The compiler may produce a warning, but it is not a compile error.

Error propagation in C is a manual convention: functions return error codes, callers check `errno`, and the discipline is maintained by the programmer. Zig's `!T` type makes this discipline non-optional.

C passes `argv` as `char**`, a raw pointer with no length information beyond `argc`. Iterating beyond `argc` is undefined behavior — there is no bounds check, no panic, no diagnostic. Zig's `argsAlloc` returns a bounded slice; out-of-bounds access in debug mode is a clean panic rather than silent memory corruption.

### Rust

```rust
// Simple form — panics on any write error
fn main() {
    println!("Hello, world!");
}

// Error-propagating form
use std::error::Error;
fn main() -> Result<(), Box<dyn Error>> {
    println!("Hello, world!");
    Ok(())
}
```

Rust's default `main()` returns `()` — void. To propagate errors, you change the return type to `Result<(), E>`, where `E` is commonly `Box<dyn Error>` for a generic error trait object. `Box<dyn Error>` requires a heap allocation per error value, because `dyn Error` is a fat pointer to a heap-allocated trait object.

Zig's error unions are zero-allocation: errors are integer tag values stored inline in the return slot. There is no trait object, no `Box`, no heap allocation in the error path. An `anyerror` value in Zig is a `u16` index into a compiler-generated table of error names.

Rust prioritizes ergonomics in `println!` — it acquires a mutex and panics on failure, because most programs do not need to handle stdout write errors. Zig prioritizes correctness uniformly: every fallible operation is `!T`, and the caller decides whether to propagate, handle, or consciously discard the error.

### Go

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, world!")
}
```

Go's `main` is minimal: it returns nothing and takes no arguments. To signal failure, you call `os.Exit(1)` or `log.Fatal(msg)`. `fmt.Println` discards write errors silently. Go's philosophy prioritizes brevity and rapid initial development; the trade-off is that error handling at the entry point is a convention rather than a type-system guarantee.

Go's runtime includes a garbage collector, goroutine scheduler, and channel implementation. These are always present, even in a hello-world binary. Zig's runtime is a thin bootstrap layer — there is no GC, no scheduler, no hidden goroutine stack pool. What you bring in explicitly through `@import` and function calls is exactly what is in your binary.

Go's `fmt.Println` performs interface dispatch at runtime using `reflect`-based type introspection. Every value you pass to `fmt.Println` is boxed into an `interface{}`. Zig's format system is entirely comptime: `stdout.print("{s}\n", .{arg})` dispatches to the correct formatter for each argument type at compile time, with zero runtime dispatch overhead.

### C++

```cpp
#include <iostream>

int main() {
    std::cout << "Hello, world!" << std::endl;
    return 0;
}
```

C++ inherits C's `int main()` contract with the same limitations. `std::endl` flushes the output buffer — a common performance pitfall, since `'\n'` alone does not flush and is often what you actually want. `operator<<` overloading can involve hidden allocations and vtable dispatch depending on the types involved; there is no guarantee that `std::cout << value` is a direct write.

`#include <iostream>` is famously expensive to parse — it can add tens to hundreds of milliseconds per translation unit because it drags in large portions of the C++ standard library headers. This is a perennial source of build-system friction in large C++ codebases. Zig's `@import("std")` adds no per-file overhead beyond the first compilation, because the parsed and analyzed result is cached and reused.

C++ exceptions from `main` that escape become `std::terminate`, typically producing no useful diagnostic. Zig's explicit error union approach ensures that errors produce their name on stderr at minimum, and enables structured error handling in `main` without any overhead in the success path.

---

## 5. Common Mistakes

### Forgetting `pub` on `main`

```zig
// WRONG — the linker cannot find a 'main' entry point
const std = @import("std");

fn main() !void {  // Missing 'pub'
    const stdout = std.io.getStdOut().writer();
    try stdout.print("Hello\n", .{});
}
```

```
error: 'main' is private
```

The compiler rejects this with a clear error. `pub` is not a default; it is an explicit declaration that a symbol is visible outside its file. Functions without `pub` exist only within the file that declares them. This is one of the first errors Zig beginners encounter, and it is worth understanding clearly: unlike Java where everything defaults to package-accessible, Zig defaults to private. You must opt in to visibility.

### Confusing `std.debug.print` with Standard Output

```zig
// This writes to STDERR, not stdout.
std.debug.print("Result: {d}\n", .{42});
```

When you redirect stdout (`./program > output.txt`), output from `std.debug.print` still appears on your terminal because it goes to stderr. If you are writing a program that produces machine-readable output — JSON, CSV, a binary protocol — **always use `std.io.getStdOut().writer()`** for the payload. `std.debug.print` is for development diagnostics and should not appear in production output paths.

### Format String Type Mismatches

```zig
const value: u32 = 42;

// WRONG — {s} expects a string slice, not an integer
try stdout.print("{s}\n", .{value});

// CORRECT
try stdout.print("{d}\n", .{value});
```

Zig's format string is validated at compile time. The specifier must be compatible with the argument type, and the compiler catches mismatches:

```
error: expected 'u32', found type '[]const u8'
```

This is a significant safety advantage over C's `printf`, where `printf("%s\n", 42)` is undefined behavior — detectable only at runtime with sanitizers enabled, or not at all. In Zig, it is a compile error.

Common format specifiers for reference:
- `{s}` — UTF-8 string slice (`[]u8`, `[]const u8`, `[:0]const u8`)
- `{d}` — decimal integer
- `{x}` / `{X}` — lowercase / uppercase hexadecimal
- `{b}` — binary representation
- `{o}` — octal representation
- `{e}` — scientific notation for floats
- `{f}` — decimal notation for floats
- `{any}` — any type, using its default formatter
- `{}` — uses the type's custom `format` method if it defines one

### Ignoring Error Return Values

```zig
pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    // Missing 'try' — compile error, not a warning
    stdout.print("Hello\n", .{});
}
```

```
error: error union is ignored
    note: consider using 'try', 'catch', or 'if'
```

`stdout.print` returns `!void`. If you discard the return value of any function that returns an error union, the compiler rejects it. This is not a warning that you can suppress; it is a hard error. You must handle it:

- `try stdout.print(...)` — propagate the error to the caller
- `stdout.print(...) catch {}` — explicitly discard the error (conscious no-op)
- `stdout.print(...) catch |err| { ... }` — handle the error inline

The explicit discard with `catch {}` documents intent: you are not forgetting the error, you are choosing to ignore it. There is no mechanism to silently ignore error unions.

### Misunderstanding `@import` Return Value

```zig
// @import returns a type (a namespace), not a runtime value
const std = @import("std");  // 'std' is a compile-time type, not a runtime object

// You cannot compute the import path at runtime
fn get_module_name() []const u8 { return "std"; }

const dynamic = @import(get_module_name());  // COMPILE ERROR
```

```
error: @import requires a string literal
```

`@import` is evaluated entirely at compile time. The path argument must be a string literal — a compile-time constant. You cannot dynamically select a module based on runtime conditions. If you need runtime dispatch between implementations, use a tagged union (Chapter 12) or an interface-like pattern with comptime duck typing (Chapter 7).

### Stack Overflow with Large Local Arrays

```zig
pub fn main() !void {
    // DANGER — 1 MB on the stack; default stack size is often only 1–8 MB
    var buffer: [1024 * 1024]u8 = undefined;
    _ = buffer;
}
```

Large arrays declared as local variables live on the stack. If the array exceeds the available stack space, the program faults with a segmentation fault — not a Zig error, not a clean panic. The check is not automatic for this class of failure in all build modes. For large buffers, use an allocator:

```zig
pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const buffer = try allocator.alloc(u8, 1024 * 1024);
    defer allocator.free(buffer);
    // Use buffer safely
}
```

A useful rule of thumb: stack arrays up to ~64 KB are generally safe. For anything larger, allocate explicitly.

### Circular Imports

```zig
// a.zig
const b = @import("b.zig");

// b.zig
const a = @import("a.zig");  // ERROR: cycle
```

```
error: import cycle detected
```

Zig detects circular imports at compile time and rejects them. Unlike C, where forward declarations in headers allow mutual dependencies between translation units, Zig's module graph must be acyclic. The solution is to extract shared types into a third file — `types.zig` or `common.zig` — that both `a.zig` and `b.zig` import, with no back-edge.

---

## 6. Performance Considerations

### Debug vs. Release: The Structural Difference

A `Debug` build and a `ReleaseFast` build of the same hello-world program behave identically from a user's perspective, but their binary structure is fundamentally different.

`Debug` mode inserts:
- Bounds checks on every slice access (`slice[i]` panics if `i >= slice.len`)
- Integer overflow checks on every arithmetic operation (addition, subtraction, multiplication)
- Null dereference checks on every optional unwrap with `orelse unreachable`
- Unreachable-branch panics that produce a stack trace and source location

Each of these adds a conditional branch and (on the error path) a call to the panic handler. For arithmetic-heavy inner loops, this overhead can be 2–5× slower than `ReleaseFast`.

`ReleaseFast` removes all safety checks, runs LLVM's full optimization pipeline (inlining, loop unrolling, vectorization, and dead code elimination), and strips safety check code entirely. The result is significantly smaller and faster at the cost of undefined behavior when bugs occur.

`ReleaseSafe` is the production recommendation: full LLVM optimization plus all safety checks retained. Bugs produce clean panics with source locations rather than silent data corruption. For most server software and CLI tools, this is the right default.

### The I/O Performance Gap

`std.debug.print` is not designed for throughput. Each call:
1. Acquires a global mutex (for thread safety)
2. Calls `write()` on the stderr file descriptor
3. Returns immediately without buffering

For a program printing thousands of lines per second, this is slow. Every print is a syscall. Use `bufferedWriter` for bulk output:

```zig
const std = @import("std");

pub fn main() !void {
    const stdout_file = std.io.getStdOut();
    var bw = std.io.bufferedWriter(stdout_file.writer());
    const stdout = bw.writer();

    for (0..10_000) |i| {
        try stdout.print("{d}\n", .{i});
    }

    try bw.flush();  // Mandatory — remaining data in the buffer must be flushed explicitly
}
```

`bufferedWriter` accumulates writes in a 4096-byte internal buffer by default, performing actual `write()` syscalls only when the buffer fills or `flush()` is called. For programs producing significant text output, this is typically a 10–50× throughput improvement on Linux and macOS.

**The flush requirement is not optional.** The buffer is stack-allocated inside `bw`; it has no destructor. When `main` returns, unflushed data is simply lost — no write happens, no error is reported. This is an explicit-is-better-than-implicit trade-off: you gain no hidden allocations and no destructor call overhead, but you must explicitly flush. For long-running programs or streaming output, call `bw.flush()` after each logical output unit.

### Compilation Speed

The self-hosted codegen backend — used for `Debug` builds on supported targets in Zig 0.14 — is significantly faster than the LLVM backend. A project that takes 30 seconds to build in `ReleaseFast` mode may take 3 seconds in `Debug` mode. For tight edit-run-debug cycles on large codebases, this difference is material.

Incremental compilation provides further savings. After the first build, only files whose ZIR has changed since the previous build are re-analyzed. In a 200-file project, modifying a single leaf file results in only that file's semantic analysis being redone; the ZIR and AIR for all other files are reused from cache. The linker still runs, but incremental linking in the self-hosted linker is also faster than a full LLD link.

### Dead Code Elimination and Binary Footprint

Zig's compilation model enables precise dead code elimination. Functions in the standard library that are not transitively called from `main` are not compiled into the binary at all. This is different from linking against `libc`, which brings in symbol tables and potentially large object files regardless of which functions you call.

```zig
const std = @import("std");

pub fn main() !void {
    // Only std.io and its dependencies are compiled into the binary.
    // std.net, std.crypto, std.json, std.Thread — none of them appear.
    const stdout = std.io.getStdOut().writer();
    try stdout.print("minimal\n", .{});
}
```

The result for this program in `ReleaseFast` mode with static musl linking is approximately 8–12 KB — smaller than many C programs that link against glibc. For embedded targets or size-constrained deployments, this property is not just a nicety but a requirement.

---

## 7. Best Practices

### Use `zig init` from the Start

Even for a program that is currently five lines, establishing the project layout immediately costs nothing and avoids restructuring later. The `build.zig` pattern gives you `zig build run` and `zig build test` from day one, and the correct directory structure scales without modification from a single-file tool to a 50-module library.

Reserve single-file `zig run` for truly throwaway scripts: one-off data transformations, quick experiments, code you will not version-control. For anything you commit to a repository, use `zig init`.

### Separate Business Logic from Entry-Point Error Handling

The idiom of a `run()` function called from `main` is the cleanest way to use `try` freely in business logic while giving `main` full control over the user experience on failure:

```zig
const std = @import("std");

pub fn main() void {
    run() catch |err| {
        const stderr = std.io.getStdErr().writer();
        stderr.print("error: {s}\n", .{@errorName(err)}) catch {};
        std.process.exit(1);
    };
}

fn run() !void {
    // Use 'try' without ceremony; all errors propagate to main's catch
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);

    const stdout = std.io.getStdOut().writer();
    try stdout.print("Arguments: {d}\n", .{args.len - 1});
}
```

`main()` returns `void` — it handles all errors internally and translates them to user-readable messages and exit codes. `run()` is free to use `try` at every level without concern for user experience. `@errorName(err)` returns the error's identifier as a `[:0]const u8` string with no allocation.

For programs where distinct error types warrant distinct messages or exit codes, use `catch |err| switch (err)` in `main` to map error values to messages explicitly.

### Use `stdout.writer()` for Program Output

The rule is simple: `std.io.getStdOut().writer()` for anything the user should see or that a pipe or redirect should capture. `std.debug.print` for temporary diagnostics you intend to remove before shipping. The discipline is easy to maintain and prevents debug output from accidentally reaching production output streams.

### Establish the Allocator Strategy in `main`

Any program that may grow to perform allocations should establish its allocator in `main` and thread it through function calls from the start:

```zig
pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer {
        const check = gpa.deinit();
        if (check == .leak) @panic("memory leak detected");
    }
    const allocator = gpa.allocator();

    try run(allocator);
}

fn run(allocator: std.mem.Allocator) !void {
    _ = allocator;  // Pass down to callees as needed
}
```

`GeneralPurposeAllocator` in `Debug` mode tracks every allocation and reports leaks with stack traces when `deinit()` is called. This catches use-after-free and double-free bugs during development at zero cost in release builds. The explicit leak check — `if (check == .leak) @panic(...)` — documents intent: you are asserting that your program manages memory correctly and want a hard failure if it does not.

### Module Boundaries from the Start

Rather than growing a monolithic `main.zig`, split by concern from the beginning. The cost of a new file is near zero:

```zig
// src/main.zig
const std = @import("std");
const config = @import("config.zig");
const processor = @import("processor.zig");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const cfg = try config.load(allocator);
    defer cfg.deinit(allocator);

    try processor.run(allocator, cfg);
}
```

Each file exposes only what is `pub`. Module boundaries enforce encapsulation without any framework, annotation system, or build-system configuration beyond the file itself. Reading `main.zig` of a well-structured Zig project tells you the program's high-level structure at a glance.

---

## 8. Examples

### Example 1: Hello World — Canonical Form

```zig
// hello.zig
// Build: zig build-exe hello.zig
// Run:   ./hello

const std = @import("std");

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();
    try stdout.print("Hello, world!\n", .{});
}
```

This is the idiomatic form. Writes to stdout. Handles write errors via `try`. Uses the correct empty argument tuple `.{}` for a format string with no specifiers.

### Example 2: Argument Echo with Index

```zig
// echo_args.zig
// Build: zig build-exe echo_args.zig
// Run:   ./echo_args foo bar baz

const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);

    const stdout = std.io.getStdOut().writer();

    // args[0] is the program name; skip it.
    for (args[1..], 0..) |arg, i| {
        try stdout.print("[{d}] {s}\n", .{ i, arg });
    }
}
```

`args[1..]` produces a slice from index 1 to the end — no off-by-one arithmetic, no raw pointer math, bounds-checked in debug mode. The indexed `for` form using `0..` avoids a separate counter variable while remaining explicit.

### Example 3: A Multi-Module Program

```zig
// src/greet.zig
// A library module: no main, no direct I/O. Pure function + test.

const std = @import("std");

/// Allocates and returns a greeting string for the given name.
/// The caller owns the returned slice and must free it.
pub fn makeGreeting(allocator: std.mem.Allocator, name: []const u8) ![]u8 {
    return std.fmt.allocPrint(allocator, "Hello, {s}!", .{name});
}

test "makeGreeting produces the correct string" {
    const result = try makeGreeting(std.testing.allocator, "Zig");
    defer std.testing.allocator.free(result);
    try std.testing.expectEqualStrings("Hello, Zig!", result);
}
```

```zig
// src/main.zig
// Build: zig build-exe src/main.zig  (or via build.zig with proper root_source_file)

const std = @import("std");
const greet = @import("greet.zig");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);

    // Use the first argument as the name, or "world" if none is given.
    const name = if (args.len > 1) args[1] else "world";

    const greeting = try greet.makeGreeting(allocator, name);
    defer allocator.free(greeting);

    const stdout = std.io.getStdOut().writer();
    try stdout.print("{s}\n", .{greeting});
}
```

Run the module's test independently:
```
$ zig test src/greet.zig
All 1 tests passed.
```

Key patterns demonstrated here:
- `greet.zig` is a pure library module: no `main`, no direct I/O, fully testable in isolation
- Allocator threading: `main` owns the allocator, `makeGreeting` receives it as a parameter, and the caller owns the returned allocation
- `defer allocator.free(greeting)` — cleanup is adjacent to acquisition, deterministic, and not dependent on destructors or scoping rules
- `std.fmt.allocPrint` — heap-allocates a formatted string, returns `![]u8`
- The `if` expression as a value: `const name = if (condition) a else b` is a single expression, not a statement

### Example 4: Graceful Error Handling in `main`

```zig
// file_info.zig
// Reports the size of a file given as a command-line argument.
// Demonstrates structured error handling and clear exit codes.
//
// Build: zig build-exe file_info.zig
// Run:   ./file_info build.zig

const std = @import("std");

const AppError = error{
    MissingArgument,
    FileTooLarge,
};

pub fn main() void {
    run() catch |err| switch (err) {
        AppError.MissingArgument => {
            std.debug.print("Usage: file_info <filename>\n", .{});
            std.process.exit(1);
        },
        AppError.FileTooLarge => {
            std.debug.print("Error: file exceeds the 1 MB processing limit.\n", .{});
            std.process.exit(2);
        },
        else => {
            std.debug.print("Unexpected error: {s}\n", .{@errorName(err)});
            std.process.exit(255);
        },
    };
}

fn run() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const args = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, args);

    if (args.len < 2) return AppError.MissingArgument;

    const filename = args[1];
    const file = try std.fs.cwd().openFile(filename, .{});
    defer file.close();

    const stat = try file.stat();
    if (stat.size > 1024 * 1024) return AppError.FileTooLarge;

    const stdout = std.io.getStdOut().writer();
    try stdout.print("'{s}' is {d} bytes ({d} KB)\n", .{
        filename,
        stat.size,
        stat.size / 1024,
    });
}
```

Key patterns:
- `pub fn main() void` — main handles all errors internally; nothing escapes to the bootstrap's generic handler
- `catch |err| switch (err)` — exhaustive pattern matching over the error set with a mandatory `else` arm for unrecognized errors
- Custom error set `AppError` — typed, documentable, distinguishable at the call site
- `@errorName(err)` — converts any error value to its string name without allocation; useful for the `else` catch-all
- `std.fs.cwd().openFile` — opens a file relative to the working directory; `defer file.close()` ensures cleanup on all paths
- `file.stat()` — returns filesystem metadata including `size: u64`
- Integer division `stat.size / 1024` — deliberate truncation, Zig requires explicit integer arithmetic

---

## 9. Summary and Exercises

### Summary

The first Zig program reveals the language's core commitments in miniature.

**`pub fn main() !void`** encodes the entry point contract. `pub` makes the symbol visible to the linker. `!void` makes error propagation a type-system concern: ignored error unions are compile errors, not runtime surprises. The entry point is a regular function — there is nothing special about it beyond the linker's expectation of the name.

**`@import`** returns a file as a compile-time struct type. There are no header files, no textual inclusion, no include guards. The standard library is accessed identically to any module you write yourself, because they are the same mechanism. The standard library is Zig code you can read.

**The compilation pipeline** runs from source → AST → ZIR → AIR → machine code, with caching at multiple stages. Debug builds use the fast self-hosted codegen backend. Release builds route through LLVM for optimization. The bootstrap in `lib/std/start.zig` wraps `main`, handling error-to-exit-code translation. You can read it.

**I/O is explicit**: there is no global `print`. You obtain a writer, use it, and flush it. `std.debug.print` writes to stderr. For throughput, wrap in `bufferedWriter` and flush explicitly.

**Errors are values**: error unions must be acknowledged. Every `try` is visible in the source. Every `catch` is a decision point. The chain from leaf function to `main` is type-checked.

The central insight of this chapter is this: **the standard library and your application code are the same thing.** `@import("std")` and `@import("my_module.zig")` are indistinguishable mechanisms. There is no privileged runtime you depend on blindly, no hidden bootstrapping you cannot inspect, no magic anywhere in the call chain. Every behavior of a Zig program traces to source code you can open and read. That transparency is not accidental. It is the design.

---

### Exercises

**Exercise 3.1: A Line-Counting Tool**

Build a command-line program `linecount.zig` that accepts one or more filenames as arguments, counts the lines in each file, and prints a summary:

```
lines.zig:    42
build.zig:    15
missing.txt:  (error: FileNotFound)
Total:        57
```

Requirements:
- Use `std.fs.cwd().openFile` and a `bufferedReader` wrapping the file's reader for efficient line-by-line reading
- Do not abort on the first missing or unreadable file — report the error per-file and continue to the next argument
- Use `std.process.argsAlloc` for arguments; exit with a usage message if no arguments are given
- Define a custom error set for application-level errors (`MissingArguments`) and translate them to user-readable messages with appropriate exit codes in `main`

This exercise forces you to practice per-item error handling (not all errors should bubble to `main`), buffered I/O, and clean separation between the error-handling layer in `main` and the logic in `run`.

**Exercise 3.2: A Modules Experiment**

Create a three-file project exploring module boundaries and allocator threading:

- `src/math.zig` — exports `pub fn fibonacci(n: u64) u64` computed iteratively (not recursively to avoid stack overflow for large `n`), plus a `test` block that verifies the first 10 Fibonacci numbers against known values
- `src/format.zig` — exports `pub fn formatFib(allocator: std.mem.Allocator, n: u64) ![]u8` that returns the string `"fib(N) = RESULT"` as an allocated slice; includes a test block using `std.testing.allocator`
- `src/main.zig` — imports both, reads `n` from the command line (parse it with `std.fmt.parseInt(u64, args[1], 10)`), and prints the formatted result

Run `zig test src/math.zig` and `zig test src/format.zig` independently to verify each module. Then `zig build-exe src/main.zig` and run the combined program.

The goal is to internalize the allocator threading pattern — `math.zig` needs no allocator, `format.zig` takes one as a parameter, `main.zig` owns the allocator — and to practice writing testable modules that are not coupled to `main`.

**Exercise 3.3: A Production-Quality Entry Point**

Take the argument echo program from Example 2 and refactor it into a production-quality CLI tool named `echo-args`:

- `main()` returns `void` and translates all errors to user-readable messages with specific exit codes: 0 for success, 1 for no arguments given, 2 for any I/O error
- All output goes through a `bufferedWriter` wrapping stdout, explicitly flushed before exit
- Implement a `--version` flag that prints `echo-args 0.1.0` and exits 0 without printing arguments
- Implement a `--count` flag that prints only the number of arguments (excluding the program name), not the arguments themselves

Implement flag parsing manually — scan the `args` slice for flags before the main loop, rather than using a library. This exercise reveals how much Zig's explicit approach costs in boilerplate, and how that cost is bounded and transparent rather than hidden in a framework.
