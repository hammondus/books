# Chapter 2: The Zig Toolchain

A language is only as usable as its toolchain. This is a lesson C learned the hard way over five decades: the language itself is portable, but the ecosystem of compilers, build systems, cross-compilation rigs, and package managers fractured into a landscape that requires significant institutional knowledge to navigate. Zig treats the toolchain not as an afterthought but as a first-class deliverable — one that ships as a single binary and covers installation, compilation, linking, formatting, testing, cross-compilation, and package management in a unified, coherent surface.

This chapter covers everything from getting Zig onto your machine to configuring a production-grade build that cross-compiles for multiple targets. The underlying build system deserves special attention: unlike Makefiles, CMake, or Cargo, `build.zig` is not a configuration file. It is a Zig program that compiles and executes at build time. Understanding that distinction — and its implications — changes how you think about build systems entirely.

---

## 1. Basic Usage

### Installing Zig

Zig distributes as a self-contained archive: a single binary (`zig` or `zig.exe`), the standard library source, and bundled libc headers. There is no installer that writes to system registries, no package of shared libraries, and no `sudo make install` ceremony. You decompress the archive, add the directory to `PATH`, and you are running.

**Direct download** from [ziglang.org/download](https://ziglang.org/download/) gives you the latest stable release (0.14.0 as of this writing) or any nightly build. Nightly builds are stable for practical work — the Zig project keeps nightly builds functional by policy.

The production-recommended installation method is **`zigup`**, a version manager analogous to `rustup` or `fnm`:

```sh
# Install zigup (the zigup binary itself is a single static binary)
# On Linux/macOS:
curl -L https://github.com/marler8997/zigup/releases/latest/download/zigup-linux-x86_64 -o zigup
chmod +x zigup
sudo mv zigup /usr/local/bin/

# Fetch and activate Zig 0.14.0
zigup 0.14.0

# Fetch and activate the latest nightly
zigup master

# List installed versions
zigup list

# Switch between installed versions
zigup 0.13.0
```

`zigup` stores Zig versions in `~/.zig/` and symlinks the active version into your `PATH`. This is essential for projects that pin a specific Zig version: `zigup` reads a `.zigversion` file in the project root and automatically switches to the pinned version when you enter the directory.

On **Windows**, Scoop provides the cleanest path:

```powershell
scoop install zig
# or for nightly:
scoop bucket add versions
scoop install zig-dev
```

The Windows Subsystem for Linux (WSL2) path works fine, but native Windows compilation with the actual Windows binary produces dramatically better cross-compilation results for Windows targets.

### Verifying the Installation

```sh
zig version
# 0.14.0

zig env
# {
#   "zig_exe": "/home/user/.zig/0.14.0/zig",
#   "lib_dir": "/home/user/.zig/0.14.0/lib",
#   "std_dir": "/home/user/.zig/0.14.0/lib/std",
#   "global_cache_dir": "/home/user/.cache/zig",
#   "version": "0.14.0",
#   "target": "x86_64-linux-gnu.2.36"
# }
```

`zig env` is immediately useful: it tells you where the standard library lives (handy when ZLS is misconfigured) and the resolved default target triple.

### The Core Commands

```sh
# Compile and run a single file, no build.zig needed
zig run src/main.zig

# Compile a single file to an executable
zig build-exe src/main.zig -O ReleaseFast

# Compile a single file to an object file
zig build-obj src/lib.zig

# Compile and run all test blocks in a file
zig test src/main.zig

# Format source files in-place
zig fmt src/

# Format check (exit code 1 if any file would change — for CI)
zig fmt --check src/

# Invoke the build system (reads build.zig)
zig build

# Invoke a named build step
zig build test
zig build run

# List all available targets
zig targets | head -30

# Use Zig as a drop-in C compiler
zig cc -o my_c_prog main.c -O2
zig c++ -o my_cpp_prog main.cpp -std=c++17
```

The `zig cc` and `zig c++` commands deserve specific attention. They expose Zig's bundled Clang frontend with a GCC/Clang-compatible flag interface. Any Makefile or CMake project that respects `CC` and `CXX` environment variables can be cross-compiled using Zig as the compiler with zero changes to the project's build files.

### A Minimal `build.zig`

For any project more complex than a single file, you will have a `build.zig` at the project root:

```zig
// build.zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    // Accept -Dtarget=<triple> and -Doptimize=<mode> from the command line
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "myapp",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // Install the artifact to zig-out/bin/
    b.installArtifact(exe);

    // Create a `zig build run` step
    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());
    if (b.args) |args| run_cmd.addArgs(args);

    const run_step = b.step("run", "Build and run the app");
    run_step.dependOn(&run_cmd.step);

    // Create a `zig build test` step
    const unit_tests = b.addTest(.{
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    const run_tests = b.addRunArtifact(unit_tests);

    const test_step = b.step("test", "Run all unit tests");
    test_step.dependOn(&run_tests.step);
}
```

With this file in place:

```sh
zig build                     # Compile → zig-out/bin/myapp
zig build run                 # Compile and run
zig build run -- --port 8080  # Pass arguments after --
zig build test                # Compile and run all test blocks
zig build -Doptimize=ReleaseFast  # Release build
zig build -Dtarget=aarch64-linux  # Cross-compile for ARM Linux
```

---

## 2. Under the Hood

### The Build System Execution Model

The most important thing to understand about Zig's build system is its execution model: **`build.zig` is not parsed as configuration; it is compiled and executed as a Zig program**. When you run `zig build`, the following sequence occurs:

1. The Zig compiler compiles `build.zig` and its dependencies into a temporary executable.
2. That executable is run, receiving `std.Build` context that describes the host environment, command-line arguments, and the build graph API.
3. The `pub fn build(b: *std.Build) void` function populates a **DAG of build steps** by calling `b.addExecutable`, `b.addTest`, `b.step`, and similar.
4. The build runner traverses the DAG, executing steps in dependency order, with parallelism where the graph allows.
5. Steps are cached using content-addressed storage: if the inputs to a step have not changed since the last build, the step is skipped.

This means that `build.zig` has the full Zig language available for build logic: loops, conditionals, comptime computation, standard library I/O, and function calls. You can read a JSON configuration file, query the filesystem, spawn child processes, and construct build steps dynamically — all at build time, in the same language as your project code.

The cache lives in `~/.cache/zig/` on Linux/macOS and `%LOCALAPPDATA%\zig\` on Windows. It is a content-addressed store keyed on the hash of all compiler inputs: source files, compiler flags, target triple, and the Zig version. A clean rebuild is triggered automatically whenever any input changes.

### The Compiler Backend Architecture

Zig supports two code generation backends:

**The LLVM backend** (`-fllvm`) is the default for release builds. It produces highly optimized machine code, benefits from LLVM's decades of optimization work, and supports every target that LLVM supports. The cost is compile-time performance: LLVM's optimization pipeline is thorough but slow, particularly at link time.

**The self-hosted x86 backend** (`-fno-llvm`, available for x86_64 targets) is Zig's own code generator, written in Zig. It prioritizes fast incremental compilation over peak optimization quality. In debug builds targeting x86_64, Zig uses this backend by default, which is why `zig run` and `zig test` feel snappy: the LLVM pipeline is bypassed entirely.

This dual-backend architecture reflects a deliberate choice: developer cycle time (compile → test → iterate) uses the fast self-hosted backend; release builds use LLVM for peak performance. The split is transparent to build.zig.

### Target Triples in Depth

Zig's target triple has the form `<arch>-<os>-<abi>`, with each component taking specific values from `zig targets`:

- **Architectures**: `x86_64`, `aarch64`, `arm`, `thumb`, `riscv64`, `wasm32`, `mips`, etc.
- **Operating systems**: `linux`, `macos`, `windows`, `freestanding`, `wasi`, `freebsd`, `openbsd`, etc.
- **ABIs**: `gnu` (glibc), `musl` (musl libc), `msvc`, `eabi` (ARM bare metal), `none`, etc.

The ABI component determines which libc implementation is linked. When the ABI is `musl`, Zig compiles musl libc from source and links it statically — the resulting binary has zero dynamic library dependencies and runs on any Linux kernel regardless of the system libc version. When the ABI is `gnu`, Zig links against glibc dynamically, requiring a compatible glibc version at runtime.

For truly portable Linux binaries, `x86_64-linux-musl` produces static binaries that run on any 64-bit Linux.

### Content-Addressed Caching

The cache invalidation model is worth understanding because it affects incremental build performance. Each artifact in the cache is identified by a hash computed from:

- The source file content hash
- The full set of compiler flags
- The target triple
- The Zig version
- Transitive hashes of all imported modules

This means that switching between targets does not invalidate the cache for the original target; both exist simultaneously. It also means that `zig build -Dtarget=aarch64-linux` and `zig build -Dtarget=x86_64-linux` produce independently cached artifacts. Large cross-compilation matrices benefit significantly from this: a CI pipeline that builds for six targets only recompiles files that have actually changed.

---

## 3. Why This Design?

### Build System as Code

The dominant paradigm in build systems is declarative configuration: CMake's `CMakeLists.txt`, Cargo's `Cargo.toml`, Meson's `meson.build`. These formats express *what* to build, but they are intentionally limited in *how* they express it. This limitation is a feature for simple cases — a `Cargo.toml` is readable by anyone — but it becomes a liability when projects need conditional compilation logic, generated sources, platform-specific linking, or multi-stage pipelines. The universal response is a proliferation of escape hatches: CMake's procedural `if/function/macro` system, Cargo build scripts (`build.rs`), Meson's Python modules.

Zig's position is that this progression is inevitable, so it should be the starting point: **the build system is always a program**. There is no small declarative format that grows into a programming language. `build.zig` is a Zig program from the first line.

The concrete advantages:

- **Type safety**: `b.addExecutable(.{ .name = "app", .root_source_file = b.path("src/main.zig") })` is type-checked. Misspelling `.root_source_file` is a compile error, not a silent ignored field.
- **Refactoring**: Common build patterns can be extracted into Zig functions. A function that sets up a test executable with standard dependencies can be called once per component. No copy-paste of CMake boilerplate.
- **Introspection**: The build script can read `builtin.zig` for target information, query the filesystem for platform-specific libraries, or fetch a version string from a file — without a secondary scripting language.
- **Cross-compilation in the build logic**: `b.standardTargetOptions(.{})` reads the `-Dtarget` flag and returns a `std.Build.ResolvedTarget`. The build script can branch on `target.result.os.tag` to include platform-specific sources without any preprocessor equivalent.

### The Single-Binary Distribution Model

Every other C/C++ toolchain distributes as a collection of binaries: `cc`, `ld`, `ar`, `as`, `ranlib`, system headers, libc. Managing these correctly across platforms requires careful packaging. Zig collapses this entire collection into one binary and one directory of source files, making the version of every component of the toolchain exactly deterministic: the version of Zig *is* the version of the compiler, the linker, and the libc headers. There are no "system headers may differ" surprises.

The bundled libc headers cover glibc, musl, mingw-w64, and the macOS SDK headers (for macOS targets, macOS SDK headers are required — Zig handles obtaining them separately via the `zig build` target machinery). For most non-Apple targets, zero external dependencies are required.

### Cross-Compilation as the Default

The design decision to make cross-compilation a single flag rather than a special mode stems from a principle: **the host and target should never be conflated**. A build that happens to target the host machine is just a cross-compilation where host == target. Treating it as a distinct case would mean maintaining two code paths in the compiler and build system, which Zig declines to do.

---

## 4. Competing Approaches

### C: Multiple Toolchains Required

Cross-compiling a C project traditionally requires a complete toolchain for each target: `aarch64-linux-gnu-gcc`, `x86_64-w64-mingw32-gcc`, etc. These must be installed and configured separately. Managing sysroots for each target — the collection of system headers and libraries compiled for that target — is a project in itself. Tools like `crosstool-ng` exist precisely because the manual process is prohibitively complex.

Zig's `zig cc` can substitute for any of these cross-compilers because Zig bundles all the necessary libc implementations and header files. A Makefile line `CC=zig cc -target aarch64-linux-musl` cross-compiles a C project with no additional installation.

### Rust / Cargo

Cargo is a well-designed build system for single-language Rust projects. Its `Cargo.toml` is readable and concise for common cases, and `build.rs` provides a Turing-complete escape hatch when needed. Cross-compilation in Rust requires installing separate targets (`rustup target add`), and C library integration requires `pkg-config` or manual configuration in `build.rs`.

Where Zig differs: `build.zig` is the same language as the project code, so there is no mental context switch between the build script and the application. Cargo's `build.rs` is Rust, but it communicates with Cargo through side-channel stdout conventions (`println!("cargo:rustc-link-lib=...")`) which are string-based and not type-checked. Zig's build API is fully typed.

### Go

Go's `go build` toolchain is famously simple and cross-compilation-capable with `GOOS` and `GOARCH` environment variables. It does not require a separate toolchain per target because Go distributes a full runtime for each target. The Go build system's limitation is its inflexibility: it does not support custom build steps, asset embedding beyond `//go:embed`, or integration with non-Go dependencies without `cgo`, which re-introduces external toolchain requirements.

Zig's build system handles arbitrary dependencies including C and C++ libraries via `addCSourceFiles` and system package integration, at the cost of more explicit configuration.

### CMake / Meson

CMake and Meson are build system generators: they produce Makefiles or Ninja files rather than directly invoking compilers. The generation layer exists for IDE integration and cross-platform compatibility, but it means there are two layers to debug when something goes wrong. Both use their own domain-specific languages with limited expressiveness (Meson's Python-like syntax is more capable than CMake's). Both require a C/C++ toolchain separate from the build system itself.

`build.zig` eliminates the generation layer: the build runner is `zig` itself, which compiles and runs `build.zig` directly. There is one tool to install and one language to understand.

---

## 5. Common Mistakes

### Pinning the Zig Version

Zig does not yet have a stable ABI or a frozen language specification (1.0 is in progress). Minor versions regularly change compiler APIs and standard library signatures. The most common newcomer mistake is not pinning the Zig version in a project.

The fix is a `.zigversion` file in the project root:

```
0.14.0
```

`zigup` reads this file when the working directory is inside the project. Additionally, `build.zig.zon` supports a `minimum_zig_version` field that causes `zig build` to emit an error if the running Zig is too old:

```zig
// build.zig.zon
.{
    .name = "myapp",
    .version = "0.1.0",
    .minimum_zig_version = "0.14.0",
    .dependencies = .{},
    .paths = .{""},
}
```

### Confusing `zig run` with `zig build run`

`zig run src/main.zig` compiles `src/main.zig` in isolation — it does not read `build.zig`, does not link any dependencies declared there, and does not apply the optimization level or target configured in the build. It is useful for quick experiments with single-file programs.

`zig build run` respects the full build graph, including linked libraries, custom compile flags, module imports, and the `-Dtarget` and `-Doptimize` options. For any non-trivial project, always use `zig build run`.

### Incorrect Target Triple Syntax

Target triples require exact spelling. Zig is not forgiving of approximations:

```sh
# Wrong — Zig uses 'aarch64', not 'arm64'
zig build -Dtarget=arm64-linux-musl   # error: unknown CPU architecture: arm64

# Correct
zig build -Dtarget=aarch64-linux-musl

# Wrong — OS must be lowercase
zig build -Dtarget=x86_64-Linux-gnu  # error

# Correct
zig build -Dtarget=x86_64-linux-gnu

# Use `zig targets` to enumerate valid values
zig targets | python3 -m json.tool | grep '"arch"' | head -20
```

`zig targets` outputs a JSON document listing every valid combination. When in doubt, query it.

### The `zig-out/` and `zig-cache/` Directories

`zig build` creates two directories in the project root:

- `zig-out/` — installed artifacts (binaries, libraries). This is what you deploy.
- `zig-cache/` — compilation cache for this specific project. This is an internal directory.

Both should be in `.gitignore`. A common mistake is committing `zig-cache/` — it is large, binary, platform-specific, and regenerated automatically. `zig-out/` should also not be committed; produced binaries belong in a release artifact store.

### ZLS Not Finding the Standard Library

ZLS (the Zig Language Server) needs to know where the Zig binary is to locate the standard library. On systems with multiple Zig versions or non-standard install paths, ZLS may fail silently or show spurious errors. The fix is explicit configuration in the editor:

For VS Code, in `.vscode/settings.json`:
```json
{
    "zig.zigPath": "/home/user/.zig/0.14.0/zig",
    "zig.zls.path": "/usr/local/bin/zls"
}
```

`zig env` gives you the correct path for `zig_exe`; use that value.

### Using `zig build-exe` for Non-Trivial Projects

`zig build-exe` is the low-level compiler invocation, analogous to calling `cc` directly. It accepts a single root source file and command-line flags for linking and optimization. It does not read `build.zig`. For any project with multiple modules, C library dependencies, or custom build logic, `zig build-exe` is the wrong tool. Use `zig build` with a `build.zig`.

---

## 6. Performance Considerations

### Incremental Compilation and the Cache

Zig's build cache is content-addressed, meaning rebuild decisions are based on input hashes rather than timestamps. This eliminates the `make` problem of timestamps being unreliable across NFS mounts, archive extraction, or manual file touches. It also means that parallel compilation and distributed caching are naturally supported: multiple machines can share a cache directory and never redundantly compile the same input combination.

In practice, for a project of moderate size (50–200 source files), a full rebuild from cache takes milliseconds. A true full rebuild (cache deleted) depends heavily on the LLVM backend: debug builds bypass LLVM and use the self-hosted backend, which is 5–10× faster. Release builds using LLVM for heavy optimization can be substantially slower.

The performance split by mode for a medium-sized project:

| Build Mode | Backend | Expected Compile Time (50 files) |
|---|---|---|
| Debug (default) | Self-hosted x86_64 | 1–3 seconds |
| ReleaseSafe | LLVM | 8–15 seconds |
| ReleaseFast | LLVM, no safety | 10–20 seconds |
| ReleaseSmall | LLVM, size optimized | 12–25 seconds |

These are rough orders of magnitude; actual times depend heavily on code complexity and hardware.

### Parallel Build Steps

The `std.Build` DAG is executed with parallelism equal to the number of logical CPUs on the host by default. The `-j` flag overrides this:

```sh
zig build -j4   # Limit to 4 parallel jobs
zig build -j1   # Serial execution (useful for debugging build output)
```

Steps that are independent in the DAG (for example, building a library and running an unrelated test suite) execute in parallel automatically. The build author does not need to annotate parallelism explicitly — the dependency graph encodes it.

### `zig fmt` Performance

`zig fmt` is an AST-based, deterministic formatter. It runs at approximately the speed of a file I/O scan: on a modern SSD, formatting 1,000 source files takes under a second. This makes it practical to run as a pre-commit hook without perceptible latency.

Unlike `clang-format`, which reformats based on token rules that can produce surprising results for complex expressions, `zig fmt` always produces output that matches what the Zig AST printer would generate. The output is fully canonical: there is exactly one correctly formatted representation of any valid Zig source file.

---

## 7. Best Practices

### Always Provide Both `-Dtarget` and `-Doptimize` Options

Use `b.standardTargetOptions` and `b.standardOptimizeOption` in every `build.zig`. They provide the standard `-Dtarget` and `-Doptimize` flags automatically, making the build immediately cross-compilation-capable and release-configurable without extra code:

```zig
pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});
    // Pass both to every addExecutable / addStaticLibrary call
}
```

### Separate Library and Executable Targets

For projects that produce a library (whether for internal use or as a published package), separate the library compilation from the executable that links it:

```zig
pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // The library — can be used by the exe and by tests independently
    const lib = b.addStaticLibrary(.{
        .name = "mylib",
        .root_source_file = b.path("src/lib.zig"),
        .target = target,
        .optimize = optimize,
    });

    const exe = b.addExecutable(.{
        .name = "myapp",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    exe.linkLibrary(lib);
    b.installArtifact(exe);

    // Tests run against the library directly, not through the executable
    const lib_tests = b.addTest(.{
        .root_source_file = b.path("src/lib.zig"),
        .target = target,
        .optimize = optimize,
    });
    const run_lib_tests = b.addRunArtifact(lib_tests);
    const test_step = b.step("test", "Run library tests");
    test_step.dependOn(&run_lib_tests.step);
}
```

This pattern keeps test compilation fast (tests link only the library, not the full executable) and makes the library independently importable by downstream projects.

### Use `b.path()` for All Source File References

Zig 0.12+ replaced anonymous struct path literals with `b.path()`, which produces a `std.Build.LazyPath`. Always use `b.path()` rather than constructing paths as strings — `LazyPath` is tracked by the build system for dependency purposes, so changes to referenced files correctly trigger recompilation.

### Editor Integration: VS Code

The `zigtools.vscode-zig` extension (available in the VS Code Marketplace) installs ZLS automatically if it is not already present. The setup is minimal:

1. Install the extension.
2. Open a folder containing a `build.zig`.
3. On first open, the extension prompts to download the correct ZLS version. Accept.

For projects using nightly Zig, ZLS nightly builds are published alongside each Zig nightly. If the version mismatch warning appears, install the matching ZLS version:

```sh
# Install ZLS matching the current Zig version
zigup fetch-zls   # if zigup supports it for your install
# or download from https://github.com/zigtools/zls/releases
```

### Editor Integration: Zed

Zed has first-party Zig support. Open Zed settings (`Cmd+,` on macOS) and ensure the `zig` language extension is installed. Zed auto-detects ZLS if it is on `PATH`. For multiple Zig versions, set the `zig_exe_path` in the Zig language settings within Zed's `settings.json`:

```json
{
  "languages": {
    "Zig": {
      "language_servers": ["zls"],
      "format_on_save": "on"
    }
  },
  "lsp": {
    "zls": {
      "settings": {
        "zig_exe_path": "/home/user/.zig/0.14.0/zig"
      }
    }
  }
}
```

### Committing a `.zigversion` File

Pin the Zig version in `.zigversion` at the project root. This single file ensures that `zigup` and any version-aware CI environment uses the correct compiler. Document the Zig version prominently in the project README.

---

## 8. Examples

### Example 1: A Complete Multi-Module Project Build

This example shows a realistic `build.zig` for a project that has a library, an executable, tests, and a benchmark target:

```zig
// build.zig — multi-module project
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // ── Core library ──────────────────────────────────────────────────────────
    const core_lib = b.addStaticLibrary(.{
        .name = "core",
        .root_source_file = b.path("src/core/lib.zig"),
        .target = target,
        .optimize = optimize,
    });

    // Expose the library's root module so the exe and tests can import it
    const core_module = core_lib.root_module;

    // ── Executable ───────────────────────────────────────────────────────────
    const exe = b.addExecutable(.{
        .name = "myapp",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    // Import the core module as "core" within main.zig
    exe.root_module.addImport("core", core_module);
    b.installArtifact(exe);

    // ── Run step ─────────────────────────────────────────────────────────────
    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());
    if (b.args) |args| run_cmd.addArgs(args);
    const run_step = b.step("run", "Build and run");
    run_step.dependOn(&run_cmd.step);

    // ── Tests ────────────────────────────────────────────────────────────────
    const core_tests = b.addTest(.{
        .root_source_file = b.path("src/core/lib.zig"),
        .target = target,
        .optimize = optimize,
    });
    const run_core_tests = b.addRunArtifact(core_tests);

    const integration_tests = b.addTest(.{
        .root_source_file = b.path("src/integration_test.zig"),
        .target = target,
        .optimize = optimize,
    });
    integration_tests.root_module.addImport("core", core_module);
    const run_integration_tests = b.addRunArtifact(integration_tests);

    const test_step = b.step("test", "Run all tests");
    test_step.dependOn(&run_core_tests.step);
    test_step.dependOn(&run_integration_tests.step);

    // ── Benchmark (ReleaseFast only) ──────────────────────────────────────────
    const bench = b.addExecutable(.{
        .name = "bench",
        .root_source_file = b.path("src/bench.zig"),
        .target = target,
        .optimize = .ReleaseFast,  // Always fast for benchmarks
    });
    bench.root_module.addImport("core", core_module);
    const run_bench = b.addRunArtifact(bench);

    const bench_step = b.step("bench", "Run benchmarks (ReleaseFast)");
    bench_step.dependOn(&run_bench.step);
}
```

### Example 2: Cross-Compilation to Multiple Targets

This `build.zig` builds the same executable for a matrix of targets, producing all artifacts in a single `zig build release` invocation:

```zig
// build.zig — multi-target release build
const std = @import("std");

const release_targets = [_]std.Target.Query{
    .{ .cpu_arch = .x86_64,  .os_tag = .linux,   .abi = .musl  },
    .{ .cpu_arch = .aarch64, .os_tag = .linux,   .abi = .musl  },
    .{ .cpu_arch = .x86_64,  .os_tag = .windows, .abi = .gnu   },
    .{ .cpu_arch = .aarch64, .os_tag = .macos,   .abi = .none  },
    .{ .cpu_arch = .x86_64,  .os_tag = .macos,   .abi = .none  },
};

pub fn build(b: *std.Build) void {
    // Default target for development
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "myapp",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    b.installArtifact(exe);

    // Multi-target release step
    const release_step = b.step("release", "Build for all release targets");

    for (release_targets) |t| {
        const resolved = b.resolveTargetQuery(t);
        const target_exe = b.addExecutable(.{
            .name = "myapp",
            .root_source_file = b.path("src/main.zig"),
            .target = resolved,
            .optimize = .ReleaseFast,
        });

        // Install each binary to zig-out/bin/<target>/myapp
        const target_str = b.fmt("{s}-{s}", .{
            @tagName(t.cpu_arch.?),
            @tagName(t.os_tag.?),
        });
        const install = b.addInstallArtifact(target_exe, .{
            .dest_dir = .{ .override = .{ .custom = target_str } },
        });
        release_step.dependOn(&install.step);
    }
}
```

Running `zig build release` produces:

```
zig-out/
  x86_64-linux/myapp
  aarch64-linux/myapp
  x86_64-windows/myapp.exe
  aarch64-macos/myapp
  x86_64-macos/myapp
```

### Example 3: Integrating a C Library

Many Zig projects need to call into existing C libraries. `build.zig` handles this directly:

```zig
// build.zig — linking a C library (libz / zlib in this example)
pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "compress_demo",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    // Link the system zlib
    exe.linkSystemLibrary("z");

    // Or link a C source file directly — no external library needed
    exe.addCSourceFile(.{
        .file = b.path("src/vendor/mylib.c"),
        .flags = &.{ "-std=c11", "-O2" },
    });

    // Add a directory of C headers
    exe.addIncludePath(b.path("src/vendor/include"));

    // Link libc explicitly when using C source files
    exe.linkLibC();

    b.installArtifact(exe);
}
```

---

## 9. Summary & Exercises

### Summary

The Zig toolchain is a deliberate departure from the "ecosystem of tools" model that C/C++ development requires. A single `zig` binary replaces the compiler, linker, assembler, formatter, test runner, build system, and cross-compilation toolchain. `build.zig` is not a configuration language — it is a Zig program that defines a typed DAG of build steps, making the build system as refactorable and type-safe as the application code.

Cross-compilation is first-class and requires nothing beyond a `-Dtarget` flag; bundled libc implementations eliminate the sysroot configuration problem entirely. The dual-backend architecture (fast self-hosted for debug, LLVM for release) keeps iteration cycles short while preserving peak performance for production builds.

ZLS provides language server features (completions, go-to-definition, rename, inline diagnostics) for both VS Code and Zed. The single recommended configuration is an explicit `zig_exe_path` matching the pinned project version.

The key insight of this chapter: **the build system being a program in the same language as the project is not a novelty — it is the correct design**. The alternative (a declarative format that eventually grows a Turing-complete extension language) produces the same outcome with more layers and more cognitive context switching.

### Exercises

**Exercise 1: Multi-Target Release Automation**
Extend the multi-target release `build.zig` from Example 2 to also produce SHA-256 checksums for each binary. Add a custom `b.step("checksums", ...)` that runs after all binaries are built, reads each binary from `zig-out/`, computes its SHA-256 using `std.crypto.hash.sha2.Sha256`, and writes a `checksums.txt` file to `zig-out/`. Implement this as a custom `std.Build.Step` subtype with a `make` function.

**Exercise 2: Build-Time Configuration Injection**
Add a custom build option using `b.option(u16, "port", "Default server port")` that injects the value into the compiled binary as a comptime constant. In `src/main.zig`, read this value using `@import("build_options").port`. Verify that changing `-Dport=9000` at build time produces a binary that uses port 9000 without any runtime argument parsing.

**Exercise 3: C Library Wrapper**
Take any small C library (libm, sqlite amalgamation, or a single-header library like `stb_image.h`) and write a `build.zig` that compiles it as a C source file, links it into a Zig executable, and calls at least one of its functions from Zig. Use `@cImport` for header translation. Verify that the resulting binary is fully statically linked (no system library dependencies) by targeting `x86_64-linux-musl` and inspecting the output with `ldd` or `file`.
