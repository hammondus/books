# Chapter 22: The Concurrency Model

Go’s reputation rests heavily on its built‑in concurrency primitives. Unlike most languages that bolt concurrency onto an existing threading model, Go embeds a runtime scheduler, lightweight goroutines, and channel-based communication directly into the language. This chapter dissects the concurrency model: how goroutines differ from OS threads, the M:N scheduler’s inner workings, why goroutines are cheap, and how traditional models compare.

---

## 1. Basic Usage

Starting a concurrent task in Go is syntactically minimal. The `go` keyword spawns a new goroutine that executes the given function concurrently with the caller.

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func say(s string) {
    for i := 0; i < 3; i++ {
        fmt.Println(s)
        time.Sleep(10 * time.Millisecond)
    }
}

func main() {
    // Launch two goroutines
    go say("world")
    go say("hello")

    // Wait enough time to see the interleaved output.
    // (In real code, use a WaitGroup instead of a fixed sleep.)
    time.Sleep(100 * time.Millisecond)
}
```

**Proper synchronisation with `sync.WaitGroup`**  
Sleeping to wait for goroutines is a bug waiting to happen. Use `WaitGroup`:

```go
func main() {
    var wg sync.WaitGroup

    // Launch a lambda as a goroutine
    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println("Hello from goroutine")
    }()

    wg.Wait()
}
```

**Starting many goroutines** – a core demonstration of their low cost:

```go
func main() {
    var wg sync.WaitGroup
    const n = 100000
    wg.Add(n)
    for i := 0; i < n; i++ {
        go func(id int) {
            defer wg.Done()
            // Simulate light work
            _ = id * id
        }(i)
    }
    wg.Wait()
    fmt.Println(n, "goroutines finished")
}
```

On a modern laptop, 100 000 goroutines start and exit within a few hundred milliseconds, consuming only a handful of OS threads.

---

## 2. Under the Hood: M:N Scheduling & The Go Scheduler

Goroutines are **not OS threads**. They are cooperatively scheduled execution contexts managed entirely by the Go runtime. The runtime uses an **M:N scheduler**: it maps M goroutines to N OS threads, where N typically equals `GOMAXPROCS` (default: number of CPU cores).

### The Three Core Structures

The scheduler relies on three main components:

- **G (Goroutine)** – A lightweight execution context. Each G contains its stack (starting ~2KB, growing as needed), instruction pointer, registers, and scheduling metadata.
- **M (Machine)** – An OS thread. The runtime creates Ms to execute Gs. An M that blocks on a syscall may be replaced by a new M so that other Gs can continue.
- **P (Processor)** – A logical processor that holds a runqueue of runnable Gs. The number of Ps is set by `GOMAXPROCS`. A P must be attached to an M to execute Gs. Ps decouple the runqueue from OS threads, enabling work stealing.

```
┌─────────────────┐     ┌─────────────────┐
│       P0        │     │       P1        │
│ local runqueue  │     │ local runqueue  │
│ [G G G ...]     │     │ [G G G ...]     │
└────────┬────────┘     └────────┬────────┘
         │                       │
    ┌────▼────┐              ┌────▼────┐
    │   M0    │              │   M1    │
    │ (thread)│              │ (thread)│
    └────┬────┘              └────┬────┘
         │                       │
    [OS scheduler]          [OS scheduler]
         │                       │
      CPU core                CPU core
```

### Work Stealing

When a P’s local runqueue is empty, it attempts to **steal** half the Gs from another P’s queue. This keeps all cores busy and dramatically reduces tail latency. If all queues are empty, the spinning Ms may eventually park the OS thread.

### Preemption

Go 1.14 introduced **asynchronous preemption**. Before that, a goroutine that never made function calls (e.g., an empty `for {}`) could lock a thread indefinitely. Now the runtime sends a signal to the thread at scheduler checkpoints (e.g., on function entry, on back edges of loops) and forces preemption. This makes even tight loops interruptible.

### Stack Management

Early Go used segmented stacks (linked list of fixed-size stack segments), which caused “hot split” problems. Since Go 1.3, stacks are **contiguous and grow dynamically**:

- Initial size: ~2KB (compared to ~1MB for a typical OS thread).
- When a stack overflow is detected, the runtime allocates a new, larger stack (typically doubling), copies the old stack content, and adjusts pointers. This is fast because only the goroutine’s own stack is touched.

**Result:** You can create millions of goroutines without exhausting memory, as long as each stack remains small.

### Syscalls and Blocking

When a goroutine performs a blocking syscall (e.g., `read` on a file), the M executing it enters a syscall state. The runtime detaches the P from that M and gives the P to a new or waiting M, allowing other goroutines to run. Once the syscall returns, the original M tries to reacquire a P; if none is available, its goroutine is parked on the global runqueue, and the M goes to sleep.

Network I/O (e.g., `net.Conn.Read`) is non‑blocking under the hood – the runtime uses an epoll/kqueue loop to wait for readiness and then resumes the blocked goroutine without tying up an M.

---

## 3. Why This Design?

The Go team had a clear philosophy: **make concurrency easy and safe for the masses**, without sacrificing performance on multi‑core machines.

### Simplicity over Async/Await

Other languages (JavaScript, Rust, Python) use async/await with explicit futures/promises. This forces color‑coded functions (async vs sync) and requires propagatings `.await` everywhere. Go’s goroutines let you write sequential‑looking code that blocks naturally, yet the runtime multiplexes it efficiently. There is no “coloured function” problem.

### Lightweight over Heavy Threads

OS threads are expensive: they consume ~1MB of stack each, require expensive context switches (trapping into the kernel), and cannot scale to hundreds of thousands without crashing the host. Go’s small‑stack, user‑space scheduler allows programmers to think in terms of **logical concurrency** – one goroutine per incoming request, per file watch, per task – without worrying about thread limits.

### CSP as a Guiding Star

Tony Hoare’s Communicating Sequential Processes (CSP) model treats concurrency as independent processes that communicate via **channels**. Go adopted this (via `go` and channels) rather than shared memory + locks. The result is a higher‑level mental model: “Don’t communicate by sharing memory; share memory by communicating.” Goroutines are the processes; channels are the communication medium.

### No Manual Thread Pools

In Java or C++, you typically use an `ExecutorService` or thread pool to avoid creating too many threads. In Go, you just `go func()`. The runtime automatically manages the mapping to OS threads, so you rarely touch `GOMAXPROCS` except for performance tuning.

---

## 4. Competing Approaches

| Language / Model | Primitive | Cost per Unit | Scheduling | Communication |
|----------------|-----------|---------------|------------|----------------|
| **Go** | Goroutine | ~2KB stack, grows | User‑space (M:N) | Channels, mutexes |
| **Java (threads)** | `java.lang.Thread` | ~1MB stack + kernel overhead | OS scheduler (1:1) | Shared memory, `java.util.concurrent` |
| **C++ (std::thread)** | `std::thread` | ~1MB stack + OS thread | OS scheduler (1:1) | Shared memory, mutexes, atomics |
| **Rust** | `std::thread` (1:1) or `tokio` tasks | ~1MB (thread) / ~few KB (task) | OS or user‑space (M:N with tokio) | Channels, async await |
| **Python** | `threading` (GIL‑bound) or `asyncio` | ~8MB stack (thread), few KB (task) | OS (threads) or user‑space (asyncio) | Queues, futures |
| **JavaScript** | Async functions, web workers | Worker process (~10MB) | Event loop (single‑threaded) | Message passing |
| **Erlang/Elixir** | Process | ~300 words + heap | User‑space (preemptive) | Message passing (mailboxes) |

**Go vs. OS threads (Java/C++):**  
OS threads are 1:1 with kernel schedulable entities. Creating 10 000 threads is impossible on most systems. Goroutines scale to millions because they are cooperatively scheduled in user space.

**Go vs. async/await (Rust/Python/JS):**  
Async/await provides M:N‑like efficiency but introduces function colouring (async functions must be called with await). Go’s goroutines are uncoloured – any function can be spawned with `go`, and blocking calls inside it do not require syntactic ceremony. The trade‑off is that Go’s blocking I/O (e.g., `conn.Read`) must be handled by the runtime’s netpoller; you cannot easily integrate third‑party async libraries without wrapping them.

**Go vs. Erlang processes:**  
Erlang processes are even more lightweight (a few hundred bytes) and are truly preemptive (reductions‑based). However, Erlang’s functional, immutable style and actor model differ from Go’s imperative CSP. Both scale extremely well, but Erlang targets fault‑tolerant telecom systems, while Go targets system software and cloud services.

---

## 5. Common Mistakes

### 1. Goroutine Leaks

A goroutine that is blocked forever (e.g., reading from a channel that never receives) will never exit, leaking its stack and referenced objects.

```go
func leak() {
    ch := make(chan int)
    go func() {
        val := <-ch   // blocks forever – no send on ch
        fmt.Println(val)
    }()
    // function returns, but the goroutine is still alive
}
```

**Detection:** Use the race detector (not for leaks), but better: use `runtime.NumGoroutine()` in tests or `pprof` goroutine profiles.

### 2. No Panic Handling

If a goroutine panics and does not recover, the entire program crashes (unless `recover` is called in a defer). Any goroutine that can panic should have a top‑level `defer recover()` if it’s not allowed to bring down the process.

```go
go func() {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("recovered panic: %v", r)
        }
    }()
    // risky code
}()
```

### 3. Assuming Deterministic Scheduling

Goroutine order is **not** guaranteed. Code like:

```go
go fmt.Println("first")
go fmt.Println("second")
```

may print in either order, or one might not finish before the program exits. Use explicit synchronisation.

### 4. Loop Variable Capture (Pre‑Go 1.22)

```go
for i := 0; i < 5; i++ {
    go func() {
        fmt.Println(i)  // all goroutines likely print the final i (5)
    }()
}
```

**Solution:** Pass `i` as a parameter or create a new variable inside the loop. Go 1.22 changed loop variable semantics to create a new instance per iteration, but relying on that across versions is risky; always pass explicitly.

### 5. Using `time.Sleep` for Coordination

Sleeping to let goroutines finish is racy. Use `WaitGroup`, channels, or `errgroup`.

### 6. Ignoring `GOMAXPROCS`

Default `GOMAXPROCS` equals CPU cores. For CPU‑bound workloads, increasing it beyond cores won’t help and may hurt due to extra scheduler contention. For I/O‑bound workloads, it may be beneficial to increase it to reduce latency – but test first.

---

## 6. Performance Considerations

### Creation Cost

- Creating a goroutine: ~`O(1)` with a small constant (a few hundred nanoseconds to a few microseconds).
- Creating an OS thread: ~`O(10µs – 100µs)` plus kernel overhead.
- Goroutine stack grows on demand, so 1 000 000 idle goroutines consume ~2GB of stack (2KB each) plus metadata.

### Scheduler Overhead

- Work stealing keeps average overhead below 5–10% for most workloads.
- **Global runqueue** contention: infrequent because each P has a local queue.
- **Syscall heavy workloads** cause more M creation/destruction. Use `runtime.LockOSThread()` only when necessary (e.g., C libraries that require thread affinity).

### Preemption Cost

Asynchronous preemption uses signals (SIGURG) which have a cost of a few microseconds. Tight loops without function calls may still see stalls. For truly real‑time needs, Go may not be suitable.

### Blocking vs. Non‑Blocking

- **Network I/O:** uses non‑blocking fd + netpoller – excellent.
- **File I/O (disk):** typically blocking syscall – will consume an M until the syscall returns, but the runtime can launch a new M. This is fine for moderate parallelism but can lead to many Ms.
- **`time.Sleep`:** efficient – the runtime parks the G and uses a timer wheel.

### Memory & GC Pressure

Each goroutine’s stack is allocated on the heap (as far as the garbage collector is concerned). Frequent creation/destruction of many short‑lived goroutines can increase GC pressure. For very high‑rate workloads (millions per second), consider a worker pool.

### Benchmark Example

```go
func BenchmarkGoroutineStart(b *testing.B) {
    for i := 0; i < b.N; i++ {
        var wg sync.WaitGroup
        wg.Add(1)
        go func() {
            wg.Done()
        }()
        wg.Wait()
    }
}
// Result (typical): ~250 ns/op on modern hardware
```

---

## 7. Best Practices (Idiomatic Go)

### 1. Keep Goroutine Lifetimes Clear

Every `go` should be accompanied by a clear way for the goroutine to exit (e.g., a `ctx.Done()` channel, a closed `done` channel, or a bounded worker termination). Document the ownership.

### 2. Use `sync.WaitGroup` for Groups of Goroutines

```go
var wg sync.WaitGroup
for _, item := range items {
    wg.Add(1)
    go func(it Item) {
        defer wg.Done()
        process(it)
    }(item)
}
wg.Wait()
```

### 3. Pass Context for Cancellation and Timeouts

```go
func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            // do work
        }
    }
}

ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
go worker(ctx)
```

### 4. Never Start a Goroutine Without Knowing How It Stops

The `go` statement itself does not panic. It’s your responsibility to ensure the goroutine eventually returns. Use linters like `govet` with `-lostcancel` to detect some leaks.

### 5. Favor Channels over Shared State When Possible

Use mutexes only for protecting caches or simple struct fields. For coordination and pipelined work, channels produce more readable flow.

### 6. Avoid `runtime.Gosched()`

`Gosched` voluntarily yields the processor. It is almost never needed in production code and often indicates a design misunderstanding.

### 7. Set `GOMAXPROCS` Deliberately

Only change it after profiling. For containerised environments, `GOMAXPROCS` should respect CPU limits (use `automaxprocs` library).

### 8. Use `go test -race` on Any Concurrent Code

The race detector is invaluable for catching unsynchronised access to shared memory. Always run it in CI.

---

## 8. Examples

### Example 1: Concurrent URL Fetcher with Cancellation

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "sync"
    "time"
)

func fetchURL(ctx context.Context, url string, wg *sync.WaitGroup, results chan<- string) {
    defer wg.Done()
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        results <- fmt.Sprintf("%s: %v", url, err)
        return
    }
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        results <- fmt.Sprintf("%s: %v", url, err)
        return
    }
    defer resp.Body.Close()
    results <- fmt.Sprintf("%s: %d", url, resp.StatusCode)
}

func main() {
    urls := []string{
        "https://google.com",
        "https://github.com",
        "https://nonexistent.example.com",
    }
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    var wg sync.WaitGroup
    results := make(chan string, len(urls)) // buffered to avoid blocking

    for _, url := range urls {
        wg.Add(1)
        go fetchURL(ctx, url, &wg, results)
    }

    // Close results channel when all fetchers finish
    go func() {
        wg.Wait()
        close(results)
    }()

    for res := range results {
        fmt.Println(res)
    }
}
```

### Example 2: Goroutine Leak Demonstration

```go
// Leaky service: starts a goroutine that never exits.
func startLeakyWorker() {
    ch := make(chan int)
    go func() {
        for {
            select {
            case v := <-ch:
                fmt.Println(v)
            // missing case for ctx.Done()
            }
        }
    }()
    // ch never receives, goroutine stuck forever
}

// Fixed version:
func startProperWorker(ctx context.Context) {
    ch := make(chan int)
    go func() {
        for {
            select {
            case v := <-ch:
                fmt.Println(v)
            case <-ctx.Done():
                return
            }
        }
    }()
}
```

### Example 3: Running Many Goroutines with Work Stealing (using a pipeline)

```go
// Generate numbers, square them, collect results.
func main() {
    const n = 1000000
    nums := make(chan int, 100)
    squares := make(chan int64, 100)

    // Generator goroutine
    go func() {
        for i := 0; i < n; i++ {
            nums <- i
        }
        close(nums)
    }()

    // Squarer goroutines (worker pool)
    var wg sync.WaitGroup
    for i := 0; i < runtime.GOMAXPROCS(0); i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for x := range nums {
                squares <- int64(x) * int64(x)
            }
        }()
    }
    // Close squares when all squarers finish
    go func() {
        wg.Wait()
        close(squares)
    }()

    // Collector
    var sum int64
    for sq := range squares {
        sum += sq
    }
    fmt.Printf("Sum of squares from 0 to %d: %d\n", n-1, sum)
}
```

---

## 9. Summary & Exercises

### Summary

- **Goroutines** are extremely lightweight execution contexts managed by Go’s runtime, not OS threads.
- The **M:N scheduler** with work stealing and asynchronous preemption efficiently maps millions of goroutines to a handful of OS threads.
- Go’s concurrency model prioritises **simplicity** (sequential‑looking code) and **CSP** (communication via channels) over explicit async/await or manual thread pools.
- Common mistakes include goroutine leaks, ignoring panics, assuming deterministic scheduling, and using sleeps for synchronisation.
- Performance is excellent for network‑bound and many CPU‑bound tasks, but high‑frequency creation of short‑lived goroutines can increase GC pressure.
- **Idiomatic Go** demands clear goroutine lifetimes, context propagation, and the use of `WaitGroup` or channels for coordination.

### Exercises

1. **Fan‑Out / Fan‑In Aggregator**  
   Write a program that launches N goroutines, where each goroutine computes the Fibonacci of a given input (large enough to be CPU‑bound, e.g., 40). Use a channel to collect results and a `context.WithTimeout` to cancel any goroutine that exceeds 200ms. Compare the total runtime when `GOMAXPROCS=1` vs `GOMAXPROCS=8`.

2. **Goroutine Leak Detector**  
   Given the following buggy code, use `runtime.NumGoroutine()` before and after to detect the leak. Then fix it without removing the `go` statement.

   ```go
   func leakyCache() {
       cache := make(map[string]string)
       ticker := time.NewTicker(1 * time.Second)
       go func() {
           for range ticker.C {
               for k := range cache {
                   delete(cache, k)
               }
           }
       }()
       // function returns, but ticker and goroutine remain
   }
   ```

   Hint: Add a way to stop the ticker and exit the goroutine.

3. **Comparing Goroutines vs. OS Threads**  
   Write two versions of a program that does 1 000 000 very short sleeps (`time.Sleep(1µs)`): one using goroutines (with a `sync.WaitGroup`) and another using OS threads (`runtime.LockOSThread` inside each goroutine to force 1:1 mapping). Measure the wall‑clock time and peak memory usage (use `runtime.ReadMemStats` or `time` command). Why does the OS thread version fail or become extremely slow beyond ~10 000 iterations? What does this tell you about Go’s concurrency model?
