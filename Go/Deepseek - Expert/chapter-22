## Chapter 22: The Concurrency Model

Go’s concurrency model is arguably its most distinctive feature. Where many languages bolt concurrency on as a library or rely on heavyweight OS threads, Go makes concurrency a first-class language construct. This chapter strips away the syntax and dives into the engine: the scheduler, the goroutine lifecycle, and the design decisions that make “just add `go`” safe for production systems with millions of tasks. We contrast the model with traditional threading, event loops, and async/await, and we lay the foundation for channels, `select`, and advanced patterns that follow.

---

### 1. Basic Usage

The `go` keyword is the sole entry point to concurrent execution in Go. It is as simple as prefixing any function or method call:

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func greet(name string) {
    fmt.Printf("Hello, %s!\n", name)
}

func main() {
    var wg sync.WaitGroup

    wg.Add(1)
    go func() {
        defer wg.Done()
        greet("Alice")
    }()

    wg.Add(1)
    go func() {
        defer wg.Done()
        greet("Bob")
    }()

    wg.Wait()
    // Without WaitGroup, main would likely exit before the goroutines run.
}
```

Important points:

- **Arguments are evaluated immediately** in the launching goroutine, before the new goroutine begins. This avoids subtle races with changing variables.
- **You must synchronize** to observe results. A `sync.WaitGroup` is the simplest mechanism when you just need to wait for completion.
- **Goroutines are not threads**; you cannot rely on thread-local storage, and the operating system has no direct visibility of them.
- A goroutine’s stack starts at **2 KiB** (as of Go 1.22) and grows as needed, allowing hundreds of thousands of goroutines in a single process.

A minimal worker pattern without channels illustrates launching a fixed pool:

```go
func worker(id int, jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    for j := range jobs {
        // simulate work
        results <- j * 2
    }
}

func main() {
    const numWorkers = 4
    jobs := make(chan int, 10)
    results := make(chan int, 10)
    var wg sync.WaitGroup

    for w := 1; w <= numWorkers; w++ {
        wg.Add(1)
        go worker(w, jobs, results, &wg)
    }

    // send jobs
    for j := 1; j <= 5; j++ {
        jobs <- j
    }
    close(jobs)
    wg.Wait()
    close(results)

    for r := range results {
        fmt.Println(r)
    }
}
```

This uses channels, but the focus is on the `go` invocation. The simplicity of adding concurrency is deliberate—there is no ceremony, no executor service to configure, no thread pool. The runtime handles the rest.

---

### 2. Under the Hood

The magic that makes goroutines cheap is the **M:N scheduler**: M goroutines are multiplexed onto N OS threads. The key players are:

- **G** – a goroutine. It holds the stack, instruction pointer, and scheduling metadata.
- **M** – an OS thread. It executes Go code on behalf of a goroutine.
- **P** – a logical processor. A `P` holds a local run queue of goroutines and is required for an `M` to execute Go code. The number of `P`’s is set by `GOMAXPROCS` (default: number of CPU cores).

**How scheduling works in a nutshell:**

1. Each `P` maintains a local run queue of runnable goroutines (up to 256 entries).
2. When an `M` is associated with a `P`, it pops a goroutine from the local queue and executes it.
3. If a goroutine blocks (e.g., on a channel, network I/O, or a syscall), the `M` parks the goroutine and picks another runnable one. For blocking syscalls, the `M` may detach from its `P` (allowing another `M` to take it) while the syscall runs.
4. **Work stealing**: if a `P`’s local queue is empty, it randomly steals half the queue from another `P`. This keeps all threads busy without central coordination.
5. **Global run queue**: goroutines that cannot be placed locally (e.g., from a newly spawned goroutine when the local queue is full) go to a global queue, which every `P` checks periodically.

**Preemption** is cooperative at the language level: a goroutine yields when it makes a function call, performs a channel operation, or explicitly calls `runtime.Gosched()`. Since Go 1.14, the runtime also employs **asynchronous preemption** based on signals: a goroutine executing a tight CPU-bound loop without any cooperative points will be preempted after a time slice (~10 ms), ensuring fairness.

**Network poller** – file descriptors are registered with the OS’s I/O multiplexer (epoll/kqueue/IOCP). A dedicated thread (the netpoller) waits for readiness. Goroutines blocked on network I/O are parked without tying up an `M`, so a single `M` can manage thousands of pending connections.

**Stack management** is dynamic. Initial goroutine stacks are tiny (2 KiB). As the stack grows, the runtime copies it to a larger contiguous memory block, adjusting all pointers that reference stack variables (made possible by precise stack maps). This avoids the segmentation that plagued older implementations and keeps allocation costs low.

Crucially, there is **no 1:1 mapping between goroutines and OS threads**; context switching between goroutines is performed in user space by the scheduler, costing tens of nanoseconds—orders of magnitude cheaper than a kernel thread context switch.

---

### 3. Why This Design?

Go’s design philosophy for concurrency stems from three practical observations at Google:

1. **Hardware is increasingly parallel**, but writing correct multi-threaded code with locks and shared memory is error-prone.
2. **The cost of an OS thread is high**—both in memory (default stack ~1 MiB) and scheduling latency. You can’t spawn 100,000 threads without crippling a machine.
3. **Event-driven programming** (callback hell, state machines) is unnatural for most developers and obscures control flow.

The Go team chose a model that is simultaneously:

- **Lightweight** – goroutines are cheap, so the programmer doesn’t have to manage thread pools.
- **Sequential in appearance** – you write blocking, synchronous code, and the runtime handles the asynchrony. This avoids “colored functions” (the function color problem) that plague languages with `async`/`await`.
- **Preemptive** with cooperative properties – the scheduler is always in control, but developers can reason locally about blocking points.
- **Inspired by CSP** (Communicating Sequential Processes) – goroutines are independent processes that share memory by communicating over channels, rather than by exposing shared state with locks. This is the North Star: “Don’t communicate by sharing memory; share memory by communicating.”

Contrast this with:

- **Java’s green threads (Project Loom)**: similar M:N scheduling, but retrofitted into a language with deep threading APIs. Go’s model was designed from the ground up.
- **Erlang’s actor model**: messages and isolated processes; Go shares memory more freely, relying on discipline and the race detector.
- **C++ and Rust**: give raw access to OS threads and async runtimes, but leave scheduling policies to the library developer. Go’s single runtime scheduler avoids fragmentation.

The overriding philosophy is **simplicity over fine-grained control**. You don’t configure thread count, scheduling policy, or stack size; you trust the runtime. This is a trade-off, but one that makes concurrent code approachable and portable.

---

### 4. Competing Approaches

To appreciate Go’s model, a side-by-side comparison with mainstream alternatives is illuminating.

| Feature                       | Go (Goroutines)                                 | Java (Virtual Threads, Loom)                     | Python (asyncio)                                | Node.js (Event Loop)                            | C++ (std::thread/async)                       |
|-------------------------------|-------------------------------------------------|--------------------------------------------------|------------------------------------------------|------------------------------------------------|-----------------------------------------------|
| **Unit of concurrency**       | Goroutine (stack ~2 KiB, user-space)            | Virtual thread (stack grows, managed by JVM)      | Coroutine (asyncio Task)                       | Callback/Promise (single-threaded event loop)  | OS thread (kernel stack ~1 MiB)               |
| **Scheduling**                | M:N, work-stealing, preemptive + cooperative    | M:N, work-stealing, preemptive (fiber)            | Cooperative, single-threaded event loop        | Cooperative, single-threaded                   | Preemptive (OS scheduler)                     |
| **Blocking I/O**              | Parking a goroutine, no OS thread consumed      | Parking a virtual thread, no OS thread consumed   | Must use `await` on async functions; no blocking | Must never block the event loop (async only)   | Blocks the OS thread; thread-per-request limited |
| **Cost of context switch**    | ~tens of ns (user-space)                        | ~hundreds of ns (JVM-managed stack)               | Function call overhead + event loop tick       | Callback invocation overhead                   | ~1-10 µs (kernel transition)                  |
| **Programming model**         | Synchronous code, blocking allowed              | Synchronous code, blocking allowed                | Async/await, `colored` functions               | Async/await, `colored` functions               | Synchronous or callback with thread pools     |
| **Memory overhead per task**  | ~2-4 KiB (starting stack)                       | ~hundreds of bytes initially (object header)      | ~hundreds of bytes (coroutine state)           | Closure object overhead                        | ~1 MiB (stack) + kernel bookkeeping           |
| **Debugging & Profiling**     | `pprof` goroutine profiles, race detector       | JFR, thread dumps, Loom-specific tooling          | `asyncio` debug mode, event loop traces         | `async_hooks`, heapdumps                        | Standard thread debuggers, TSan               |

**Go’s advantage**: the programming model remains simple—you write straight-line code and never annotate functions as `async`. Every function can be invoked via `go` without changing its signature. That eliminates the viral async propagation and the “what color is your function?” problem.

**Java’s virtual threads** share this advantage but still live in an ecosystem where libraries may use `synchronized`, thread locals, and stack traces that assume OS threads. Go had the luxury of designing the standard library and culture around goroutines from day one.

**Node.js and Python’s asyncio** achieve high concurrency on a single thread, but any CPU-bound task blocks the entire event loop. Go’s preemptive scheduler allows CPU-intensive goroutines to run truly in parallel across multiple OS threads.

**C++ and Rust** can achieve Go-like performance with custom schedulers (e.g., Boost.Fiber, tokio), but require careful selection of runtime, lack a unified concurrency story, and can fragment the ecosystem. Go provides a single, canonical scheduler that every library shares—no need to decide between async runtimes.

The key takeaway: Go trades off the ultimate control of thread affinity or custom scheduling for a universal, easy-to-reason-about concurrency model that works out of the box.

---

### 5. Common Mistakes

Even seasoned engineers trip over goroutine fundamentals. The most frequent traps:

**1. Forgetting to wait.**
The program exits as soon as `main` returns, killing all goroutines. Always synchronize with `sync.WaitGroup`, `errgroup`, or channel completion signals.

```go
go doWork() // Oops, main may exit immediately
```

**2. The loop variable capture classic.**
Before Go 1.22, loop variables were reused across iterations. Capturing them by closure leads to all goroutines seeing the last value. (Go 1.22+ makes loop variables per-iteration, but many codebases still need the explicit capture for older versions.)

```go
for _, v := range items {
    go func() {
        process(v) // v may be the last value in Go <1.22
    }()
}
// Fix (still defensive even post-1.22):
for _, v := range items {
    v := v
    go func() { process(v) }()
}
```

**3. Unbounded goroutine creation.**
Spawning a goroutine for every incoming request or queue item without a bound leads to memory exhaustion. Use a semaphore (buffered channel or `golang.org/x/sync/semaphore`) or a worker pool for limits.

**4. Goroutine leaks.**
A goroutine that blocks forever on a channel or a mutex and is never unblocked becomes a leak. Common causes: writing to an unbuffered channel with no reader, waiting on a `select` with no case that triggers, or a producer that outlives the consumer. Use `context.Context` with deadlines or cancellation to avoid eternal waits.

**5. Assuming deterministic ordering.**
There is no guaranteed order of goroutine execution, not even with respect to the spawning goroutine. A `go` statement does not imply the new goroutine starts immediately. Any dependency on execution order is a race.

**6. Misusing `time.Sleep` for synchronization.**
Using `time.Sleep` to “wait” for a goroutine is fragile and slow. It either flushes out races only by luck or introduces latency. Always use proper synchronization primitives.

**7. Sharing data without synchronization.**
Though channels are the preferred communication, many developers fall back to shared memory with mutexes and forget to guard access. Always run the race detector (`go test -race`) to catch these.

---

### 6. Performance Considerations

Understanding the overhead of goroutines informs real-world design:

- **Creation cost**: allocating a goroutine involves a small stack (2 KiB), a `g` struct (~400 bytes), and some scheduler bookkeeping. Creating a goroutine is on the order of **~1-2 µs**, compared to ~10-50 µs for an OS thread on Linux. A million goroutines might consume ~2 GiB of memory just for stacks, but that’s often less than the equivalent in threads.
- **Context switch**: switching between goroutines on the same `M` takes roughly **50-100 ns**. In contrast, a kernel thread context switch costs **1-10 µs** due to syscall overhead, TLB flushes, and mode switches.
- **Work-stealing overhead**: stealing costs a few atomic operations; the random probing avoids centralized contention. The system scales nearly linearly with `GOMAXPROCS`.
- **Blocking syscalls**: when a goroutine enters a blocking syscall (e.g., file I/O without the netpoller), the `M` that is executing it is blocked, reducing available parallelism. The runtime will spin up new `M`s (up to a limit) to keep `P`s busy. For network I/O, the netpoller avoids this entirely.
- **GC impact**: many goroutines mean many live stack roots. The garbage collector scans goroutine stacks concurrently, but an excessive number of goroutines with deep stacks can increase GC pause time (though typically still sub-millisecond). Profile under realistic loads.
- **Cache locality**: goroutines that are scheduled on different `M`s may cause cache thrashing if they share data. Limiting concurrency with a worker pool can improve CPU cache efficiency by reusing the same OS threads and reducing migrations.

A common benchmark: comparing an I/O-bound server using goroutines vs. a thread-per-connection model. At 10,000 concurrent connections, the thread-per-connection server may consume >10 GiB of stack memory and exhibit high scheduling overhead, while the goroutine server uses a few hundred MiB and the scheduler keeps all cores busy.

**Profiling tip:** the `pprof` goroutine profile (`debug/pprof/goroutine`) shows stack traces of all goroutines, which is invaluable for spotting leaks. The scheduler tracing with `GODEBUG=schedtrace=1000` prints work-stealing statistics.

---

### 7. Best Practices

Writing idiomatic concurrent Go goes beyond syntax:

- **Let the caller control concurrency.** Library code should avoid spawning goroutines implicitly. The function that calls your code should decide whether to run something in a goroutine. This avoids hidden resource consumption and makes the behavior predictable.

- **Use `sync.WaitGroup` for simple fan-out.** When you just need to launch a batch of goroutines and wait for all to finish, a `WaitGroup` is idiomatic and lightweight.

- **Prefer `context.Context` for cancellation and deadlines.** Every goroutine that could block on I/O, channels, or time should accept a `ctx` and honor its cancellation. This is the standard mechanism to cleanly tear down goroutines. (We cover `context` thoroughly in Chapter 25.)

- **Name goroutines for debugging.** While Go doesn’t have a built-in goroutine name, you can use `runtime/pprof` labels to annotate goroutine profiles:

```go
pprof.Do(ctx, pprof.Labels("worker", "crawler"), func(ctx context.Context) {
    // all goroutines spawned within this context carry the labels
})
```

This improves visibility in production profiles.

- **Keep goroutine logic simple.** A goroutine should do one thing and communicate results via channels. Long-lived goroutines with complex state machines are harder to reason about and test. Compose simple workers.

- **Don’t spin-wait.** Never write a busy loop like `for !condition {}` inside a goroutine; it will burn CPU and prevent cooperative scheduling from yielding. Use channels, condition variables, or `time.Ticker` for waiting.

- **Limit concurrency with a semaphore pattern.** Use a buffered channel to cap the number of concurrent operations:

```go
sem := make(chan struct{}, maxConcurrency)
for _, item := range items {
    sem <- struct{}{} // acquire
    go func(item Item) {
        defer func() { <-sem }() // release
        process(item)
    }(item)
}
// Wait for all to finish (e.g., using WaitGroup + closing sem)
```

This pattern is far more flexible than a fixed thread pool.

- **Understand when blocking is okay.** In Go, a goroutine that blocks on a channel or network operation yields automatically and costs only its stack. Don’t fear blocking; it’s how the scheduler knows to do other work. Spinning or polling, on the other hand, is harmful.

---

### 8. Examples

**Example 1: Concurrent word counter using mutex.**
Demonstrates goroutines, WaitGroup, and a shared map guarded by a mutex—a deliberate use of shared memory as a baseline, which we will later replace with channels.

```go
func concurrentWordCount(files []string) map[string]int {
    var mu sync.Mutex
    counts := make(map[string]int)
    var wg sync.WaitGroup

    for _, f := range files {
        wg.Add(1)
        go func(file string) {
            defer wg.Done()
            localCounts := countWordsInFile(file) // returns map[string]int
            mu.Lock()
            for word, n := range localCounts {
                counts[word] += n
            }
            mu.Unlock()
        }(f)
    }
    wg.Wait()
    return counts
}
```

We capture `f` by function argument to avoid the loop variable bug. The mutex is held only briefly to merge maps, minimizing contention. The same task is elegantly solved with channels in the next chapter.

**Example 2: A graceful shutdown with context.**
Showing how a goroutine can be cancelled cleanly:

```go
func monitor(ctx context.Context, interval time.Duration) {
    ticker := time.NewTicker(interval)
    defer ticker.Stop()
    for {
        select {
        case <-ctx.Done():
            log.Println("monitor shutting down")
            return
        case t := <-ticker.C:
            log.Println("health check at", t)
            // perform check
        }
    }
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    go monitor(ctx, 2*time.Second)
    // ... do work, then cancel triggers cleanup
}
```

This pattern ensures no goroutine leaks.

**Example 3: Bounded parallelism with errgroup.**
The `golang.org/x/sync/errgroup` provides error propagation and context integration:

```go
g, ctx := errgroup.WithContext(ctx)
g.SetLimit(10) // max concurrent goroutines
for _, url := range urls {
    url := url
    g.Go(func() error {
        return fetchAndProcess(ctx, url)
    })
}
if err := g.Wait(); err != nil {
    log.Fatal(err)
}
```

The `errgroup` manages a WaitGroup, limits concurrency, and cancels the derived context on first error. It’s a production-ready building block.

---

### 9. Summary & Exercises

Go’s concurrency model rests on three pillars: lightweight goroutines, a work-stealing M:N scheduler, and a philosophy that encourages communication over shared state. The `go` keyword hides enormous runtime sophistication, allowing you to write straightforward synchronous code that scales to millions of concurrent tasks. Mastery begins with understanding how the scheduler works, why blocking is not a sin, and how to avoid the classic goroutine leaks and races.

**Key takeaways:**
- Goroutines are user-space threads managed by the Go runtime, with tiny stacks and cheap context switches.
- The scheduler multiplexes goroutines onto OS threads using `P`’s, work stealing, and asynchronous preemption.
- The design enables “blocking” I/O without wasting threads, sidestepping the function coloring problem of `async/await`.
- Proper synchronization, context propagation, and concurrency limits are essential for production correctness.

#### Exercises

1. **Build a bounded parallel executor.**
   Implement a function `ParallelMap(slice []T, fn func(T) R, concurrency int) []R` that applies `fn` to each element using at most `concurrency` goroutines, preserving order. Do not use third-party libraries. Measure the speedup compared to sequential execution for a CPU-bound task. How does performance scale with `concurrency` beyond `GOMAXPROCS`?

2. **Goroutine leak detector for tests.**
   Write a helper that records the number of goroutines before a test and asserts that no goroutines leaked afterward. Use `runtime.NumGoroutine()` and optionally profile stack traces to identify the leaking goroutine. Integrate it into a table-driven test suite for a custom worker pool. What patterns caused leaks in your implementation?

3. **Memory footprint comparison.**
   Create a benchmark that spawns 100,000 goroutines, each blocked on a `sync.WaitGroup`, and measures the memory usage (via `runtime.ReadMemStats`). Compare this to an equivalent thread-per-task implementation (using C or Java if you have the environment). Document the difference in RSS and virtual memory. What does this tell you about designing a connection-per-goroutine server?
