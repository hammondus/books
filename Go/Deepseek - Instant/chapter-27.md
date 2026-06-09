## Chapter 27: File & OS Operations

File I/O and operating system interactions are foundational to production services—whether you’re writing configuration files, processing logs, or orchestrating subprocesses. Go’s `os`, `io`, and `os/exec` packages provide a thin, composable layer over the underlying system calls, designed with the same “less is more” philosophy that governs the rest of the language.

This chapter assumes you already understand file descriptors, permissions, and process fundamentals. We’ll focus on how Go surfaces these concepts, the trade-offs it makes, and the patterns that avoid subtle production failures.

---

### 1. Basic Usage

Go’s file operations revolve around `*os.File`, which implements `io.Reader`, `io.Writer`, and `io.Closer`. Environment variables are accessed via `os.Getenv` / `os.LookupEnv`, and subprocesses are managed with `os/exec`.

#### Opening and Reading a File

```go
package main

import (
    "fmt"
    "io"
    "os"
)

func main() {
    f, err := os.Open("/etc/os-release")
    if err != nil {
        fmt.Fprintf(os.Stderr, "failed to open file: %v\n", err)
        os.Exit(1)
    }
    defer f.Close()

    // Read entire file (small files only!)
    data, err := io.ReadAll(f)
    if err != nil {
        fmt.Fprintf(os.Stderr, "failed to read: %v\n", err)
        os.Exit(1)
    }
    fmt.Printf("content:\n%s\n", data)
}
```

#### Writing with Explicit Permissions

```go
func writeConfig(path string, content []byte) error {
    // O_CREATE | O_WRONLY | O_TRUNC, permissions 0644 (rw-r--r--)
    f, err := os.OpenFile(path, os.O_CREATE|os.O_WRONLY|os.O_TRUNC, 0644)
    if err != nil {
        return fmt.Errorf("open config file: %w", err)
    }
    defer func() {
        if closeErr := f.Close(); closeErr != nil && err == nil {
            err = fmt.Errorf("close config: %w", closeErr)
        }
    }()

    n, err := f.Write(content)
    if err != nil {
        return fmt.Errorf("write config: %w", err)
    }
    if n != len(content) {
        return fmt.Errorf("short write: wrote %d of %d bytes", n, len(content))
    }
    return nil
}
```

#### Environment Variables

```go
func getEnvWithDefault(key, defaultValue string) string {
    if val, ok := os.LookupEnv(key); ok {
        return val
    }
    return defaultValue
}

// Example: reading a required variable
func mustGetEnv(key string) string {
    val, ok := os.LookupEnv(key)
    if !ok {
        panic(fmt.Sprintf("required environment variable %s not set", key))
    }
    return val
}
```

#### Spawning a Subprocess

```go
cmd := exec.Command("ffmpeg", "-i", "input.mp4", "output.mp3")
cmd.Stdout = os.Stdout
cmd.Stderr = os.Stderr
if err := cmd.Run(); err != nil {
    fmt.Printf("command failed: %v\n", err)
}
```

---

### 2. Under the Hood

The `os` package is a thin, platform-agnostic wrapper over the operating system’s syscall layer. When you call `os.Open`, Go invokes `syscall.Open` (on Unix) or `CreateFile` (on Windows), then wraps the resulting file descriptor in an `os.File`.

#### File Descriptors and the Runtime Poller

Each `os.File` holds an integer file descriptor (fd). On Unix, reads and writes are **blocking** syscalls. Go’s runtime avoids tying one OS thread per blocked operation by using the **netpoller** – the same mechanism used for network sockets. When you perform `Read` on a regular file, the poller is **not** engaged (because regular files always return immediately from read/write – they don’t block indefinitely). However, for **character devices** (e.g., `/dev/tty`), pipes, or FIFOs, the poller is used to make the operation compatible with goroutine scheduling.

Concretely: `os.File` distinguishes between “non-blocking-ready” files (sockets, pipes) and regular files. For regular files, the goroutine that calls `Read` may still block the underlying OS thread if the kernel I/O takes time (e.g., reading from a slow USB drive). Go’s scheduler can’t preempt a syscall that runs for a long time – that goroutine “locks” the thread. This is rarely an issue in practice, but it means you cannot rely on the scheduler to magically interleave hundreds of slow file reads without additional goroutines.

#### Memory Mapping

For high-performance random access or large files, Go provides `golang.org/x/exp/mmap` (experimental) or you can call `syscall.Mmap` directly. The standard library’s `os.File` does **not** automatically use `mmap`. If you need zero-copy reads from a file, you must use `mmap` explicitly – which maps the file into the process’s virtual address space, bypassing the read buffer and page cache overhead.

#### Environment Variable Implementation

Environment variables are stored in a process-local copy inherited from the parent. `os.Setenv` modifies this copy; it does **not** affect the parent process. Internally, Go caches environment strings in a `map[string]string` after converting from the platform’s native representation (on Unix, `environ` array of `key=value`). `os.LookupEnv` is O(1) amortized.

#### Subprocess Execution

`exec.Cmd` forks (on Unix) or creates a new process (on Windows). It sets up pipes for stdin/stdout/stderr, duplicates file descriptors, and calls `execve`. Go’s runtime ensures that after `fork`, the child process only runs async-signal-safe code. The parent goroutine uses the netpoller to wait for the child’s exit status (`wait4` or `WaitForSingleObject`). This is why `cmd.Wait` is cancellable via `Context` – the runtime registers a callback.

---

### 3. Why This Design?

The `io.Reader` and `io.Writer` interfaces are the central abstractions. Instead of providing special-purpose file methods (e.g., `readLines`, `writeString`), Go forces composition: wrap a file with `bufio.Reader`, `gzip.Reader`, or `io.LimitReader`. This design stems from the **“simple interfaces, powerful composition”** philosophy:

- **One way to read:** `Read(p []byte) (n int, err error)` works for files, network connections, in-memory buffers, and compressed streams. No method overloading, no “read with timeout” variants – you build those externally.
- **Error handling is explicit:** No exceptions that can be silently ignored. You must check `err != nil` after every read or write.
- **Defer for cleanup:** Go’s `defer` makes resource release easy, but it also exposes a subtlety: closing a file can fail (network filesystems, full disk). The standard library’s `Close` methods return errors, but many developers ignore them. This is a conscious trade-off – common case is fast and readable, but the sharp edge remains.

**Why no `try-with-resources` or `with` statement?** Go’s answer is `defer`. It works with any resource, not just files. However, it lacks automation for the “close may fail” problem – you must write the explicit `defer func() { ... }()` pattern shown earlier.

**Why does `os.Open` not support context cancellation?** Because regular file I/O cannot be interrupted reliably across all platforms. Cancelling a `read` that has already entered the kernel would require non-portable signals or forcing the file descriptor to non-blocking mode (which regular files do not support). The team’s position: if you need cancellable file I/O, use a separate goroutine and close the file or use `os.File.SetDeadline` on platforms that support it (e.g., `EPOLLRDHUP` on Linux is for sockets only). For regular files, the recommended pattern is:

```go
type cancellableReader struct {
    *os.File
    cancel context.CancelFunc
}

func (r *cancellableReader) Read(p []byte) (int, error) {
    // read in a goroutine, abort if context done
    ...
}
```

But this is complex and rarely needed. The design accepts that file I/O is not cancellable at the syscall level, so the API doesn’t pretend.

---

### 4. Competing Approaches

| Language | File I/O Model | Strengths | Trade-offs |
|----------|----------------|-----------|-------------|
| **Python** | Built‑in `open` + `with` statements, `pathlib` high‑level paths, `os` module for low‑level | Very ergonomic, automatic closing, rich path manipulation | GIL affects threaded I/O; async file I/O requires `aiofiles` or `anyio`. |
| **Java** | `java.nio.file.Files` (static helpers), `FileChannel`, `AsynchronousFileChannel` (true async) | Mature non‑blocking file APIs, memory mapping built‑in, path abstraction (`Path`) | Verbose, checked exceptions, heavy runtime. |
| **Rust** | `std::fs` returning `Result`, `tokio::fs` for async, ownership guarantees | Zero‑cost abstraction, resource safety at compile time, async support | Steep learning curve; explicit `.await` propagation. |
| **JavaScript (Node)** | `fs.promises` with async/await, `fs.createReadStream` | Non‑blocking by default, event‑driven | Single‑threaded event loop can be blocked by large file operations (requires streams). |

**Go’s distinguishing choices:**

- **No built‑in async file API** – You use goroutines, but regular file I/O is still blocking at the OS thread level. Node.js can handle many concurrent file operations without extra threads because libuv uses a thread pool. Go’s runtime does not provide a similar thread pool for file I/O automatically – you must manage your own worker pool if you have many slow file operations.
- **Explicit permission bits** – `os.OpenFile` requires the `os.O_*` flags and a permission mode. This exposes Unix semantics directly, even on Windows (where permissions are emulated). By contrast, Python’s `open` hides modes behind a string (`'w'`).
- **No `pathlib` clone** – Go’s `path/filepath` provides functions, not a fluent object API. This is consistent with Go’s “function over objects” bias. You manipulate strings; the package handles OS-specific separators and volume names.

---

### 5. Common Mistakes

#### 5.1 Leaking File Descriptors

```go
// BAD: no defer Close, or Close ignored after early return
f, err := os.Open("data.txt")
if err != nil {
    return err
}
if someCondition {
    return nil  // fd leak!
}
defer f.Close() // too late, already leaked in someCondition
```

**Fix:** Always `defer f.Close()` immediately after a successful open, before any conditional branches.

#### 5.2 Ignoring `Close` Errors

On NFS, FUSE, or full disks, `Close` may flush buffered data and fail. You must handle it:

```go
func writeAndClose(f *os.File, data []byte) (err error) {
    if _, err = f.Write(data); err != nil {
        return err
    }
    if err = f.Close(); err != nil {
        return err
    }
    return nil
}
```

#### 5.3 Assuming Short Reads/Writes Won’t Happen

`f.Read` is allowed to return fewer bytes than requested, even if EOF is not reached. This is not just for sockets – regular files on some filesystems (e.g., FUSE) can return short reads. Always loop:

```go
func readExactly(f *os.File, buf []byte) error {
    n := 0
    for n < len(buf) {
        nn, err := f.Read(buf[n:])
        if err != nil && !errors.Is(err, io.EOF) {
            return err
        }
        if nn == 0 {
            return io.ErrUnexpectedEOF
        }
        n += nn
    }
    return nil
}
```

Similarly, `Write` may return `n < len(p)` without an error. Always check.

#### 5.4 Path Injection with Environment Variables

```go
// DANGEROUS
filePath := os.Getenv("CONFIG_DIR") + "/config.yaml"
f, _ := os.Open(filePath) // if CONFIG_DIR is "../../etc", you've escaped
```

Use `filepath.Join` which sanitizes and resolves relative paths, but still trust the environment only after validation.

#### 5.5 Goroutine Leaks with `exec.CommandContext`

`cmd := exec.CommandContext(ctx, "sleep", "1000")` cancels the context – but the subprocess might not terminate immediately. Go will send `SIGKILL` on Unix only when the context is cancelled **and** you’ve set `cmd.Wait`. However, if you forget to call `Wait`, the process becomes a zombie. Always call `Wait`.

#### 5.6 Using `os.Stat` Before File Operations

```go
// BAD: TOCTOU race
if _, err := os.Stat(filename); err == nil {
    f, err := os.Open(filename) // file could have been deleted or changed
}
```

The only safe pattern is to open and then handle the error.

---

### 6. Performance Considerations

#### Syscall Overhead

Every `Read` and `Write` on an unbuffered file is a syscall. For small operations (e.g., reading one byte at a time), this kills performance. Use `bufio.NewReader(f)` or `io.CopyBuffer` with a reasonably sized buffer (e.g., 32KB).

#### Allocation Patterns

`ioutil.ReadFile` (deprecated) and `io.ReadAll` allocate a byte slice that grows via `append`. For large files (hundreds of MB), this can double memory usage. Prefer `io.Copy` to a known destination or use `os.File.Read` in a fixed buffer.

#### Memory Mapping for Random Access

If you need to read specific offsets repeatedly, `mmap` (via `golang.org/x/exp/mmap` or `syscall.Mmap`) can be faster than many `ReadAt` syscalls. But beware: mapped pages are not counted in Go’s memory stats, and unmapping is tricky.

#### Directory Traversal

`filepath.WalkDir` (Go 1.16+) is more efficient than `filepath.Walk` because it avoids `os.Lstat` for each entry when the directory entry provides type information. Use `WalkDir` for all new code.

#### Buffered vs. Direct I/O

Go does not expose `O_DIRECT` (Linux direct I/O) through the `os` package. You would need `syscall.Open` with `syscall.O_DIRECT`. This is rarely necessary except for bypassing the page cache in database-like engines.

---

### 7. Best Practices

1. **Always close files with defer** – and handle the close error in functions that return errors.

2. **Use `os.CreateTemp` for temporary files** – it respects the system temp dir and avoids naming collisions.

   ```go
   f, err := os.CreateTemp("", "myapp-*.json")
   if err != nil {
       return err
   }
   defer os.Remove(f.Name()) // clean up
   defer f.Close()
   ```

3. **Compose readers and writers** – To count bytes written: `countingWriter := io.MultiWriter(f, os.Stdout)`. To limit read size: `r := io.LimitReader(f, maxBytes)`. Do not write custom loops when the standard library already has them.

4. **Prefer `os.LookupEnv` over `os.Getenv`** – The latter returns an empty string for unset variables, which is ambiguous if the variable legitimately has an empty value.

5. **Use `filepath.Join` for path construction** – It handles OS separators correctly and cleans `..` and `.` when possible.

6. **Cancellable commands** – Always use `exec.CommandContext` and check `ctx.Err()` after `Wait`:

   ```go
   ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
   defer cancel()
   cmd := exec.CommandContext(ctx, "long-running")
   if err := cmd.Run(); err != nil {
       if ctx.Err() == context.DeadlineExceeded {
           // handle timeout specifically
       }
   }
   ```

7. **File permission constants** – Use `0o644`, `0o755` (Go 1.13+) for readability, not decimal numbers.

8. **Sync rarely** – `f.Sync()` forces a flush to stable storage; it’s expensive. Only call it when durability is required (e.g., database WAL).

---

### 8. Examples

#### Example 1: Safe File Writer with Atomic Replacement

```go
// WriteFileAtomically writes data to a temp file and renames it.
// This avoids partial writes being read by other processes.
func WriteFileAtomically(filename string, data []byte, perm os.FileMode) error {
    dir := filepath.Dir(filename)
    tmpFile, err := os.CreateTemp(dir, ".tmp-"+filepath.Base(filename))
    if err != nil {
        return fmt.Errorf("create temp file: %w", err)
    }
    tmpName := tmpFile.Name()
    defer func() {
        // Clean up temp file on error
        _ = os.Remove(tmpName)
    }()
    defer tmpFile.Close()

    if _, err := tmpFile.Write(data); err != nil {
        return fmt.Errorf("write temp: %w", err)
    }
    if err := tmpFile.Sync(); err != nil {
        return fmt.Errorf("sync temp: %w", err)
    }
    if err := tmpFile.Close(); err != nil {
        return fmt.Errorf("close temp: %w", err)
    }
    if err := os.Chmod(tmpName, perm); err != nil {
        return fmt.Errorf("chmod: %w", err)
    }
    if err := os.Rename(tmpName, filename); err != nil {
        return fmt.Errorf("rename: %w", err)
    }
    return nil
}
```

#### Example 2: Line-by-Line Processing with Context

```go
func ProcessLines(ctx context.Context, filename string, handler func(line string) error) error {
    f, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer f.Close()

    scanner := bufio.NewScanner(f)
    // Increase buffer if lines are long (default 64KB)
    scanner.Buffer(make([]byte, 0, 1024*1024), 2*1024*1024)

    for scanner.Scan() {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
        }
        if err := handler(scanner.Text()); err != nil {
            return fmt.Errorf("handling line: %w", err)
        }
    }
    return scanner.Err()
}
```

#### Example 3: Environment Configuration Loader

```go
type Config struct {
    Port    int
    DataDir string
    Debug   bool
}

func LoadConfigFromEnv() (*Config, error) {
    cfg := &Config{
        Port:    8080,
        DataDir: "/var/lib/app",
    }
    if portStr, ok := os.LookupEnv("APP_PORT"); ok {
        port, err := strconv.Atoi(portStr)
        if err != nil {
            return nil, fmt.Errorf("invalid APP_PORT: %w", err)
        }
        cfg.Port = port
    }
    if dir, ok := os.LookupEnv("APP_DATADIR"); ok {
        cfg.DataDir = dir
    }
    if debug, ok := os.LookupEnv("APP_DEBUG"); ok {
        cfg.Debug = debug == "1" || strings.EqualFold(debug, "true")
    }
    return cfg, nil
}
```

#### Example 4: Parallel File Checksum (Worker Pool)

```go
func ChecksumFiles(ctx context.Context, filenames []string) (map[string]string, error) {
    type result struct {
        name string
        sum  string
        err  error
    }
    results := make(chan result, len(filenames))
    var wg sync.WaitGroup

    for _, name := range filenames {
        wg.Add(1)
        go func(fname string) {
            defer wg.Done()
            h := sha256.New()
            f, err := os.Open(fname)
            if err != nil {
                results <- result{name: fname, err: err}
                return
            }
            defer f.Close()
            if _, err := io.Copy(h, f); err != nil {
                results <- result{name: fname, err: err}
                return
            }
            results <- result{name: fname, sum: hex.EncodeToString(h.Sum(nil))}
        }(name)
    }

    go func() {
        wg.Wait()
        close(results)
    }()

    sums := make(map[string]string)
    for res := range results {
        if res.err != nil {
            return nil, fmt.Errorf("processing %s: %w", res.name, res.err)
        }
        sums[res.name] = res.sum
    }
    return sums, nil
}
```

---

### 9. Summary & Exercises

Go’s file and OS packages are intentionally low-level, exposing the operating system’s semantics with minimal abstraction. By mastering `io.Reader`/`Writer` composition, proper error handling (including close errors), and the nuances of `exec.CommandContext`, you can build robust production tooling. The key takeaways:

- Always handle `Close` errors in functions that return errors.
- Buffered I/O is mandatory for performance; compose `bufio` with your files.
- Environment variables require `LookupEnv` to distinguish empty from unset.
- Subprocesses demand context deadlines and explicit `Wait`.
- Atomic file writes are best achieved via temp files and `os.Rename`.

#### Exercises

**Exercise 1: Tail with Follow**  
Implement a `tail -f`-like function in Go. Given a filename, print new lines as they are appended to the file. Handle log rotation (when the file is moved or truncated). Your implementation should detect inode changes on Unix (using `os.Stat` and `syscall.Stat_t`). Write a test that writes to a file in a goroutine while your tail function reads.

**Exercise 2: Parallel grep with Cancellation**  
Build a command-line tool that searches a set of files for a regex pattern. Use a worker pool (like the checksum example). The tool must support context cancellation (e.g., on `SIGINT`). Print matching lines with filenames. Measure performance against `grep -r` on a large source tree and discuss the differences.

**Exercise 3: Production Environment Validator**  
Write a package that validates the execution environment: writable temporary directory, available disk space (>1GB), required environment variables present, and ability to spawn a subprocess (e.g., `echo "ok"`). Return detailed error messages if any check fails. Your validator should be usable as a library in a service’s `main` function before starting the main logic. Use `syscall.Statfs` on Unix to check disk space.

**Exercise 4: Atomic Configuration Updater**  
Create a `SafeConfig` type that holds a `map[string]string` and persists to a JSON file. Implement an `Update(fn func(map[string]string) error) error` method that reads the current config, calls `fn` on an in-memory copy, then writes the updated map atomically to disk (using the atomic writer pattern). Ensure concurrent calls to `Update` are serialized with a mutex. Write a fuzz test that simulates many simultaneous updates and process crashes (by calling `os.Exit` in a test subprocess) to verify that the JSON file never becomes corrupted.
