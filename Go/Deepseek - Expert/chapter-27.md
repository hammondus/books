# Chapter 27: File & OS Operations

Go’s standard library treats the operating system as a first-class concern, but not through layers of abstraction. The `os` and `os/exec` packages provide direct, procedural access to files, directories, environment variables, and processes. For a seasoned engineer, this chapter moves past “how to open a file” and dives into the runtime’s interaction with file descriptors, the trade-offs in the process model, and the idiomatic patterns that keep production code robust without unnecessary ceremony.

## 1. Basic Usage

At the surface, Go’s file and OS operations mirror the simplicity of the language itself. You open a file, check for an error, and operate on the returned handle. There are no resource constructors, no `using` blocks, and no RAII — just an explicit `Close()` that you must not forget.

```go
f, err := os.Open("data.txt")
if err != nil {
    // Handle the error. os.IsNotExist may be relevant.
}
defer f.Close()

buf := make([]byte, 1024)
n, err := f.Read(buf)
if err != nil && err != io.EOF {
    // Handle read error.
}
```

For whole-file reads and writes, Go 1.16 introduced `os.ReadFile` and `os.WriteFile`, which handle open/close internally and return a byte slice or error:

```go
data, err := os.ReadFile("config.json")
if err != nil {
    return fmt.Errorf("reading config: %w", err)
}
```

Directory operations follow the same pattern. `os.Mkdir` creates a single directory, while `os.MkdirAll` creates the full path and is analogous to `mkdir -p`. To read directory contents, `os.ReadDir` returns a slice of `fs.DirEntry`, which provides name, type, and file info without an extra `stat` call:

```go
entries, err := os.ReadDir(".") // since Go 1.16
if err != nil {
    return err
}
for _, e := range entries {
    fmt.Println(e.Name(), e.IsDir())
}
```

Environment variables are accessed through simple lookup functions. `os.Getenv` returns an empty string for missing keys, while `os.LookupEnv` separates existence from value:

```go
val, ok := os.LookupEnv("APP_MODE")
if !ok {
    val = "development"
}
```

To execute external commands, use `os/exec`. The `Command` function builds a `Cmd` struct that is intentionally shell‑free by default, avoiding injection pitfalls. A basic invocation captures combined output:

```go
cmd := exec.Command("git", "rev-parse", "--short", "HEAD")
out, err := cmd.Output()
if err != nil {
    // non-zero exit or start failure
}
```

With context support, you can enforce deadlines and cancellation:

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()
cmd := exec.CommandContext(ctx, "ffmpeg", "-i", "input.mp4", "-c:v", "libx264", "output.mkv")
if err := cmd.Run(); err != nil {
    if errors.Is(err, ctx.Err()) {
        // timed out or cancelled
    }
}
```

## 2. Under the Hood

The `os.File` struct is a thin wrapper around a platform file descriptor — an `int` on Unix or a `syscall.Handle` on Windows. Internally, the `os` package holds a pointer to a `file` unexported struct that contains the descriptor, a name, and synchronization primitives to guarantee thread‑safe reads and writes.

When you call `f.Read()`, Go enters a syscall directly. On Linux, this means `syscall.Read(fd, p)`. Critically, regular files do **not** participate in the runtime’s network poller (epoll/kqueue/IOCP). The network poller works for sockets and pipes because those file descriptors support non‑blocking I/O and notification; regular files always appear “ready” from the kernel’s perspective, making poll‑based asynchronous I/O meaningless. As a result, a goroutine that blocks on a file read or write will sit in a blocking syscall, effectively occupying an OS thread.

The Go scheduler handles this by ensuring that the `P` (logical processor) is handed off to another `M` (OS thread) if the current `M` becomes blocked, and a new `M` may be created if none are idle. This is efficient for occasional file I/O, but when hundreds of goroutines simultaneously perform large file reads, the runtime can spawn many OS threads, increasing memory and scheduling overhead. On Windows, Go currently uses synchronous I/O for regular files as well, though the OS itself supports overlapped (asynchronous) operations; this is a deliberate trade‑off for simplicity and portability.

File descriptor management also involves a finalizer. The `os.File` struct registers a finalizer with the runtime that closes the underlying descriptor if the user hasn’t explicitly called `Close()`. Relying on the finalizer is dangerous — it may run arbitrarily late, or not at all during the program’s lifetime, leading to descriptor exhaustion. The finalizer exists only as a safety net, not a primary cleanup mechanism.

Environment variables live in a process‑wide string slice that mirrors the OS environment block. On Unix, Go copies the `environ` pointer at startup and replaces the whole slice on `Setenv` to maintain thread safety for reads, but `Setenv` itself is not safe for concurrent use. The `os.Environ()` function returns a snapshot of the current environment; modifying that slice has no effect on the live environment.

The `os/exec` package is built on `os.StartProcess`, which calls platform‑specific primitives: `fork`/`exec` on Unix, `CreateProcess` on Windows. On Unix, the runtime holds a global `ForkLock` during the fork phase to prevent other threads from being in a state that would crash or corrupt memory after a fork (for example, if a thread holds a mutex when the child process attempts to acquire it). After the fork, the parent continues, and the child process is monitored via `os.Process.Wait`, which reaps the child and collects its exit status. Failing to call `Wait` after a `Start` results in a zombie process on Unix, though the `Cmd` struct’s finalizer will attempt to `Wait` if the user forgot. Again, do not rely on finalizers.

## 3. Why This Design?

The Go team chose a procedural, minimalist approach to OS interaction instead of an object‑heavy resource hierarchy. There is no `FileReader` class, no `Directory` abstraction, no abstract base class for file systems (until the later addition of `io/fs` as a separate, optional interface). The philosophy is rooted in two principles: **the OS is already a well‑defined abstraction**, and **composability through interfaces trumps deep taxonomy**.

A file handle in Go is a concrete `*os.File`, but it implements `io.Reader`, `io.Writer`, `io.Closer`, and `io.Seeker`. This means that any function expecting a reader — a JSON decoder, a checksum computer, a network proxy — can accept a file without needing to know it’s a file. The separation of path manipulation (`path/filepath`) from I/O (`os`) keeps each package focused: `filepath` deals with string operations and platform‑specific separators, while `os` deals with syscalls.

Environment variables are a simple string‑based map, not a hierarchical configuration store. This reflects the Unix tradition and avoids the temptation to build elaborate configuration DSLs into the language runtime. The addition of `os.LookupEnv` acknowledges that “not set” and “set to empty” are distinct states, without introducing option types or nullable pointers.

For process execution, the deliberate separation of `os/exec` from `os` signals that launching subprocesses is a distinct operation with its own risks. The `Cmd` struct is configured via fields, not through a builder pattern, and the default is shell‑off, which prevents injection by construction. You must explicitly use `cmd.Shell` or invoke a shell if you need that behavior, making the security implications obvious.

The overall effect is that Go code interacting with the OS looks like a set of transparent system calls wrapped in error checks. There is no magic, no implicit cleanup, and no framework that hides the cost of I/O — a design that aligns with Go’s goal of making the cost of operations visible.

## 4. Competing Approaches

When you come from a language like **Python**, the comparison is immediate. Python’s `open()` built‑in and `pathlib.Path` offer high‑level, object‑oriented convenience: `Path.read_text()` hides the open/close cycle and exceptions. Go’s equivalent, `os.ReadFile`, is a function, not a method, and the absence of exceptions forces explicit error handling. Python’s `os.environ` is a mutable dict; Go’s is an opaque string slice with thread‑safety limitations. The most significant difference is that Python’s subprocess module can default to `shell=True`, whereas Go’s `exec.Command` never invokes a shell implicitly.

**Java** developers will find familiar concepts in `java.nio.file.Files` — a utility class with static methods like `readAllLines` and `createDirectory`. Java’s `Path` is an immutable abstraction, whereas Go uses plain strings and `filepath.Join`. Go has no checked exceptions, so every error must be inspected. Java’s `ProcessBuilder` mirrors Go’s `exec.Cmd` configuration, but Go adds native context cancellation, which in Java requires manual thread interruption.

In **Rust**, the `std::fs` and `std::process` modules feel conceptually closest to Go’s approach because both languages avoid exceptions and rely on `Result`/`error` returns. Rust’s `File::open` returns `Result<File>`, and the `?` operator propagates errors similarly to Go’s `if err != nil`. However, Rust’s ownership model ensures that a `File` is closed at drop time (RAII), eliminating the risk of descriptor leaks without a finalizer. Go’s GC‑based cleanup is less deterministic. Rust’s `std::fs::read` and `write` are equivalent to Go’s `os.ReadFile`/`WriteFile`. For processes, Rust’s `std::process::Command` is a builder with chained methods, whereas Go’s `Cmd` uses struct field assignment. Both support timeouts via dedicated APIs; Go uses `exec.CommandContext`.

**C** programmers will recognize Go’s `os` package as a safer, GC‑protected wrapper over `fopen`, `read`, `write`, and `fork/exec`. The absence of manual memory management for buffers is a relief, but the responsibility to close descriptors remains. Go’s `os.Args` and environment handling are direct analogues to the C runtime.

## 5. Common Mistakes

**Forgetting to close files** is the cardinal sin. A deferred `f.Close()` is the standard remedy, but even experienced engineers sometimes place `defer` after error‑return checks where a nil file might still be closed. This is safe because `f` is nil only on error, and `Close` on a nil `*os.File` panics — so the defer must follow the nil check:

```go
f, err := os.Open(name)
if err != nil {
    return err
}
defer f.Close()
// Correct: defer happens only after confirming f is non‑nil.
```

**Ignoring the error from Close()** is another trap. For writable files, the OS may buffer data and only discover a write error (e.g., disk full) during the `close` syscall. Closing with `defer f.Close()` silently discards that error. The robust pattern is to check the error on close when writing:

```go
defer func() {
    cerr := f.Close()
    if cerr != nil && err == nil {
        err = cerr
    }
}()
```

**Misusing file permission bits** occurs frequently. Go’s `os.FileMode` interprets a numeric constant like `0644` as an octal literal in Go source. Writing `644` (decimal) sets utterly wrong permissions. Always use `0` prefix. Additionally, the OS applies a `umask` to new files, which Go does not compensate for — it’s an OS‑level filter. Expecting `0777` to result in world‑writable files is incorrect if the process’s umask is set.

**Leaking goroutines with `exec.Cmd`** happens when `StdoutPipe` or `StderrPipe` are used but the pipes are not drained before `Wait`. The `Wait` call closes the pipes after the command exits, but if the command produces enough output to fill the pipe buffer before it finishes, the command will block, and `Wait` will never return. Always copy from pipes in a separate goroutine before calling `Wait`:

```go
stdout, _ := cmd.StdoutPipe()
cmd.Start()
go io.Copy(os.Stdout, stdout)
cmd.Wait() // Deadlock if stdout pipe is not drained concurrently.
```

**Using `os.Setenv` concurrently** is a race condition. The standard library does not synchronize writes. If one goroutine calls `os.Setenv` while another calls `os.Getenv`, the result may be a torn read or a crash. Pass environment to subprocesses via `cmd.Env` instead of modifying the global environment.

**Assuming `os.IsNotExist` covers all not‑found cases** leads to brittle error handling. An `*os.PathError` may wrap errors that are not about file existence (e.g., permission denied). Always inspect the error with `errors.Is(err, os.ErrNotExist)` for precise detection.

## 6. Performance Considerations

File I/O in Go has a straightforward cost model: each `Read` or `Write` call translates to a syscall. Unbuffered I/O on a small slice incurs the syscall overhead repeatedly. The `bufio` package amortizes this by reading large blocks from the file and serving small requests from an in‑memory buffer. For sequential scanning of line‑oriented text, `bufio.Scanner` is idiomatic and performant.

`os.ReadFile` is convenient but memory‑intensive: it allocates a byte slice sized to the entire file. For a 1 GB file, that’s a 1 GB allocation. Production services that process large logs should instead use `io.Copy` or `bufio.Reader` to stream data in fixed chunks, keeping memory bounded.

Directory traversal with `filepath.Walk` calls `os.Lstat` on every entry, which is a syscall per file. For large directories, `filepath.WalkDir` (Go 1.16) reduces syscalls because `DirEntry` already contains file type information obtained from the directory entry itself, avoiding a separate stat. This is measurable on NFS filesystems where stat calls are expensive.

Process creation via `os/exec` is inherently costly: a fork/exec involves duplicating the page table, setting up file descriptors, and loading a new binary. If your application spawns a short‑lived command in a hot path, consider a long‑running worker subprocess that you communicate with via stdin/stdout or a local socket, reusing it across requests. Context cancellation is cheap, but the process itself will consume resources until the kill signal is delivered and reaped.

Environment variable access is essentially a linear scan of a string slice (or a map lookup in Go’s internal copy). Calling `os.Getenv` inside a tight loop is fine, but `os.Setenv` replaces the entire slice on each call, incurring allocations and copy overhead. Avoid using `os.Setenv` as a configuration mechanism; instead, parse environment once at startup into a struct.

File descriptor limits are a real constraint. Each open `*os.File` consumes one descriptor. If you handle thousands of concurrent requests each opening a file, you may exhaust the process limit. Use a semaphore (buffered channel) to bound the number of concurrently open files, or restructure to use `io/fs.FS` with `os.DirFS` for read‑only access, which may open on demand but still respect limits.

## 7. Best Practices

**Open and close with discipline.** Prefer `os.Open` for reading, `os.Create` for writing (truncates), and `os.OpenFile` with flags for append (`os.O_APPEND|os.O_CREATE|os.O_WRONLY`) or read‑write. Always defer close, and check the error on close for writable files.

**Use the newer convenience functions** `os.ReadFile`, `os.WriteFile`, and `os.ReadDir` when their semantics match your needs. They reduce boilerplate and internalize error handling for the full lifecycle. For temporary files, `os.CreateTemp` is safer than hand‑rolled names.

**Construct paths with `filepath.Join`**, not string concatenation. This handles separators and platform differences. For cross‑platform compatibility, use `filepath.FromSlash` and `filepath.ToSlash` when ingesting paths from user input or URLs.

**Parse environment variables at startup** into a configuration struct. Use `os.LookupEnv` for mandatory variables and provide defaults for optional ones. Avoid spreading `os.Getenv` calls throughout the codebase; it hinders testability. To test, set `cmd.Env` when invoking subprocesses, and design your functions to accept a configuration parameter instead of reading the global environment directly.

**Prefer `exec.CommandContext` for subprocesses.** It ties process lifetime to a context, enabling automatic cleanup on timeout or cancellation. When you need to stream input or capture output concurrently, follow this safe pattern:

1. Create the command with `exec.CommandContext`.
2. If streaming output, obtain pipes with `cmd.StdoutPipe()` (and optionally `StderrPipe`).
3. Start the command with `cmd.Start()`.
4. Launch goroutines that read from the pipes and copy to your desired writers (or collect into bytes.Buffer). These goroutines should close a sync.WaitGroup or send on a channel when done.
5. Call `cmd.Wait()`. This will block until the process exits and all pipe copies have finished (because the pipes are closed by the OS only when the process exits).
6. After `Wait`, combine the pipe read errors with the command error as needed.

Do not call `cmd.Run()` if you’ve set `StdoutPipe` or `StderrPipe` — `Run` will deadlock because it internally calls `Wait` after `Start` without draining the pipes.

**Avoid global environment mutations.** Instead of calling `os.Setenv`, set `cmd.Env` to an explicit slice built from `os.Environ()` plus any overrides. This isolates subprocess environments and keeps the parent’s environment unchanged for other goroutines.

**Use `io/fs` for read‑only file trees.** Starting with Go 1.16, `os.DirFS(".")` returns an `fs.FS` that allows algorithms like `fs.WalkDir` and `fs.Glob` to operate on a directory tree abstractly. This is invaluable for testing, because you can substitute `fstest.MapFS` to simulate files in memory.

**Handle signals for graceful shutdown.** The `os/signal` package lets you trap `SIGINT` and `SIGTERM` to close resources and finish in‑flight work. This topic spills into process management; the idiomatic way is to use a cancelable context derived from `signal.NotifyContext`.

## 8. Examples

### Robust Configuration Reader

This example reads a configuration file path from an environment variable, falls back to a default, opens the file, and decodes JSON. It wraps errors with context to aid debugging.

```go
func LoadConfig(ctx context.Context) (*Config, error) {
    path := os.Getenv("CONFIG_PATH")
    if path == "" {
        path = "./config.json"
    }

    data, err := os.ReadFile(path)
    if err != nil {
        if errors.Is(err, os.ErrNotExist) {
            return nil, fmt.Errorf("config file %s not found", path)
        }
        return nil, fmt.Errorf("reading config %s: %w", path, err)
    }

    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        return nil, fmt.Errorf("parsing config %s: %w", path, err)
    }
    return &cfg, nil
}
```

### Streaming Output of a Subprocess with Timeout

This function runs a command with a 5‑second deadline, streaming stdout to a provided writer, collecting stderr separately, and returning a combined error if anything fails.

```go
func RunWithTimeout(ctx context.Context, w io.Writer, name string, args ...string) error {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    cmd := exec.CommandContext(ctx, name, args...)
    cmd.Stderr = &stderrBuf

    stdout, err := cmd.StdoutPipe()
    if err != nil {
        return fmt.Errorf("stdout pipe: %w", err)
    }

    if err := cmd.Start(); err != nil {
        return fmt.Errorf("start %s: %w", name, err)
    }

    // Copy stdout concurrently to avoid blocking the command.
    copyDone := make(chan error, 1)
    go func() {
        _, err := io.Copy(w, stdout)
        copyDone <- err
    }()

    waitErr := cmd.Wait()
    copyErr := <-copyDone

    if copyErr != nil {
        return fmt.Errorf("copy stdout: %w", copyErr)
    }
    if stderrBuf.Len() > 0 {
        // Attach stderr to the error if needed.
        if waitErr != nil {
            waitErr = fmt.Errorf("%w, stderr: %s", waitErr, stderrBuf.String())
        }
    }
    return waitErr
}
```

### Concurrent Directory Hash with Bounded Parallelism

This function walks a directory tree, computes SHA256 hashes of each file, and writes a manifest. It uses a worker pool limited to 10 open files to respect file descriptor limits.

```go
func HashDirectory(root string) (map[string]string, error) {
    hashes := make(map[string]string)
    var mu sync.Mutex

    // Bounded goroutines with a channel as semaphore.
    sem := make(chan struct{}, 10)
    var wg sync.WaitGroup

    err := filepath.WalkDir(root, func(path string, d fs.DirEntry, err error) error {
        if err != nil {
            return err
        }
        if d.IsDir() {
            return nil
        }
        wg.Add(1)
        sem <- struct{}{} // acquire
        go func(p string) {
            defer wg.Done()
            defer func() { <-sem }() // release

            data, rerr := os.ReadFile(p)
            if rerr != nil {
                // In real code, handle errors via a channel.
                fmt.Fprintf(os.Stderr, "read %s: %v\n", p, rerr)
                return
            }
            h := sha256.Sum256(data)
            hex := fmt.Sprintf("%x", h)

            mu.Lock()
            hashes[p] = hex
            mu.Unlock()
        }(path)
        return nil
    })

    wg.Wait()
    return hashes, err
}
```

## 9. Summary & Exercises

Go’s OS packages offer a procedural, transparent interface to the underlying operating system. The key takeaways:

- **Files** are simple handles that implement standard interfaces; always close them explicitly and check close errors for writes.
- **Directories** are operated on with functions like `os.MkdirAll` and `os.ReadDir`; `filepath.WalkDir` is efficient for traversal.
- **Environment variables** are a plain map; avoid global mutations and parse them once at startup.
- **Processes** are spawned safely with `exec.CommandContext`; always drain pipes before `Wait` and respect context cancellation.

### Exercises

1. **Write a file‑based key‑value store.** Design a package that persists a map of string keys to byte slices on disk. Each key is a file in a directory, with the value as file content. Ensure concurrent access is safe, handle file descriptor limits, and implement a method to compact the store by removing keys whose value size is zero. Use `os.OpenFile` with appropriate flags and `os.Remove` for deletion. Provide context‑aware expiration for stale keys.

2. **Build a controlled subprocess runner.** Create an HTTP server that accepts a JSON body with a command and a list of arguments, runs the command under a strict 2‑second timeout, restricts its environment to a whitelist of variables, and returns the stdout and stderr separated. If the command times out, kill the process group. Use `exec.CommandContext` and `syscall.SysProcAttr` to set the process group on Unix. The server must handle concurrent requests safely and bound the number of simultaneous subprocesses to 5.

3. **Implement a log tailer with file rotation detection.** Write a program that continuously tails a log file (like `tail -F`). It should detect when the file is rotated (inode change) and seamlessly switch to the new file. Use `os.Stat` to compare inode via `os.SameFile` (or the `os.FileInfo`’s `Sys()` on Unix). Handle `io.EOF` correctly, sleep when no new data is available, and support graceful shutdown via a context. Ensure no data is lost during rotation by reading any remaining bytes from the old file before switching.
