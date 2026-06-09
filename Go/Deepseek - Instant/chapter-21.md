# Chapter 21: Profiling & Optimization

## 1. Basic Usage

Go ships with a built‑in profiling and tracing suite that integrates directly into the testing and runtime toolchains. The three primary tools are **CPU profiling**, **memory profiling** (heap allocations), and **execution tracing**.

```go
// cpu_profile.go – Generate a CPU profile for an HTTP server
package main

import (
    "log"
    "net/http"
    _ "net/http/pprof" // registers pprof handlers on /debug/pprof
    "runtime"
    "time"
)

func main() {
    // Enable mutex and block profiling (optional, adds overhead)
    runtime.SetMutexProfileFraction(5)   // 1/5 of mutex events sampled
    runtime.SetBlockProfileRate(1000)     // sample blocking events every 1µs

    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()

    // Simulate work
    for i := 0; i < 10; i++ {
        go heavyAllocation()
    }
    select {}
}

func heavyAllocation() {
    for {
        _ = make([]byte, 1024*1024) // 1MB allocation per loop
        time.Sleep(100 * time.Millisecond)
    }
}
```

```bash
# Capture a 30-second CPU profile
go tool pprof -http=:8080 http://localhost:6060/debug/pprof/profile?seconds=30

# Capture a heap profile (in-use space)
curl -o heap.pb.gz http://localhost:6060/debug/pprof/heap

# Capture an allocation profile (cumulative allocations)
curl -o allocs.pb.gz http://localhost:6060/debug/pprof/allocs

# Interactive command-line pprof
go tool pprof heap.pb.gz
(pprof) top5
(pprof) list heavyAllocation
(pprof) web   # generates a Graphviz diagram
```

**Benchmark‑driven profiling** (the most common workflow):

```go
// bench_test.go
package main

import (
    "bytes"
    "testing"
)

var globalResult []byte // prevent compiler elimination

func BenchmarkConcat(b *testing.B) {
    parts := []string{"a", "b", "c", "d", "e"}
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        var buf bytes.Buffer
        for _, p := range parts {
            buf.WriteString(p)
        }
        globalResult = buf.Bytes()
    }
}

// Run with:
// go test -bench=. -benchmem -cpuprofile=cpu.out -memprofile=mem.out -blockprofile=block.out
```

**Execution tracing** (goroutine scheduling, syscalls, GC events):

```go
// trace_demo.go
package main

import (
    "os"
    "runtime/trace"
    "sync"
)

func main() {
    f, _ := os.Create("trace.out")
    defer f.Close()
    trace.Start(f)
    defer trace.Stop()

    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            _ = make([]byte, 10<<20) // allocate 10MB
        }(i)
    }
    wg.Wait()
}
```

```bash
go run trace_demo.go
go tool trace trace.out   # opens interactive web viewer
```

---

## 2. Under the Hood

### How `pprof` Works

The `runtime/pprof` package and `net/http/pprof` hook into the Go runtime’s internal sampling mechanisms:

- **CPU profiling**: The runtime sets a OS timer (SIGPROF on Unix, akin to `setitimer(ITIMER_PROF)`). Every 100 Hz (default, configurable via `runtime.SetCPUProfileRate`), the signal handler interrupts the current goroutine and records the program counter stack. The sample count is attributed to the function at the top of the stack. After the profiling period, pprof reconstructs a call graph.
- **Heap profiling**: The memory allocator (`malloc.go`) maintains a sampling rate – by default, one sample per 512 KB allocated (can be changed with `runtime.MemProfileRate`). Every allocation that triggers a sample records the stack trace at the allocation site. The profile includes `alloc_objects`, `alloc_space` (cumulative), `inuse_objects`, `inuse_space` (live at profile time).
- **Mutex profile**: Records contention events when a goroutine blocks on a mutex for longer than a configurable delay. The `SetMutexProfileFraction` sets the fraction of mutex events to sample (1 = every event, high overhead).
- **Block profile**: Similar to mutex, but records blocking on channel operations, select statements, and condition variables.

The profiles are encoded as **protocol buffers** (the `.pb.gz` format). `go tool pprof` transforms them into the classic Google pprof format (text, graph, flame graph).

### Execution Tracer Internals

The tracer (`runtime/trace`) emits a stream of timestamped events:

- Goroutine creation / start / blocking / unblocking / syscall entry‑exit
- GC phases (mark termination, sweep, etc.)
- Network poller events
- User‑annotated regions (`trace.WithRegion`)

Events are written to a binary format with a **time series encoding** (varint + delta compression). The trace viewer reconstructs a Gantt‑style view of the M:N scheduler: how goroutines were multiplexed onto OS threads, which logical processors (Ps) ran them, and where they blocked.

**Overhead**: The tracer adds ~10–30% CPU overhead and several microseconds per event. Not for production always‑on, but invaluable for short captures.

### Benchmark Compiler Optimizations

`go test -bench` runs the target function `b.N` times. The compiler is **extremely aggressive** about eliminating dead code. Without `globalResult`, the benchmark would be a no‑op. The assignment to a package‑level variable tricks the compiler into believing the result is observable.

```go
var sink *T   // alternative: store pointer to heap

func Benchmark(b *testing.B) {
    for i := 0; i < b.N; i++ {
        sink = allocateSomething()
    }
}
```

`-benchmem` prints allocation stats: `B/op` (bytes allocated per iteration) and `allocs/op` (number of allocations). These come from the memory profiler sampled during the benchmark.

---

## 3. Why This Design?

The Go team embedded profiling and tracing into the runtime rather than leaving it to external tools for several reasons:

1. **Zero‑cost abstraction (when off)**: Profiling hooks are conditional – when the sampling rate is zero (`MemProfileRate = 0`), the allocator runs with no extra branches. CPU profiling uses an OS signal that is entirely disabled when not active. This aligns with Go’s philosophy: you pay only for what you use.

2. **Self‑describing binaries**: A Go binary contains all runtime and symbol information. You can `go tool pprof` a running production service without installing debug symbols separately. This is a deliberate contrast with C++ (where you often need DWARF debuginfo packages) or Java (where you rely on JVM TI agents).

3. **Standardization across all Go code**: Because every Go program uses the same runtime, `pprof` and `trace` work identically for a CLI tool, an HTTP server, or a gRPC service. The lack of fragmentation reduces cognitive load – a huge win for operations teams.

4. **Benchmarking as a first‑class citizen**: The `testing` package’s benchmark support (`*testing.B`) forces developers to write reproducible, isolated performance tests. Go rejects the “profile in production only” mindset; you can (and should) profile during development with the same tools used in production.

5. **No external agents**: Java’s `jstack` or Python’s `py-spy` require separate processes and often impose overhead. Go’s `net/http/pprof` is a regular HTTP handler – you can enable it behind an auth middleware and scrape profiles without installing extra daemons.

**Why premature optimization is dangerous** – this is explicitly called out in the Go proverbs: “Don’t guess about performance.” By providing easy, high‑resolution profiling, the Go team encourages a data‑driven workflow: measure first, then optimize.

---

## 4. Competing Approaches

| Language/Tool | Profiling Mechanism | Philosophy |
|---------------|---------------------|-------------|
| **Go (pprof)** | Built into runtime, sampling‑based, self‑describing | “Low friction, data‑driven optimization. Use the same tool from dev to prod.” |
| **Java (JFR, async‑profiler)** | JVM Flight Recorder – low‑overhead, continuous; async‑profiler uses `perf_events` for wall‑clock profiling | “Continuous production profiling with security and historical buffers.” |
| **Python (cProfile, py‑spy)** | Deterministic profiling (function call counting) or sampling via `py‑spy` (attaches to running process) | “Python’s dynamic nature makes sampling harder; cProfile adds high overhead.” |
| **Rust (perf, `flamegraph`, `criterion`)** | Relies on Linux `perf` for CPU profiles; `criterion` for benchmarking; `pprof‑rs` output | “You build the tooling from ecosystem crates; no single blessed profiling story.” |
| **C++ (perf, VTune, valgrind)** | OS‑level sampling (`perf`) or instrumented binaries (`gperftools`) | “Unmatched depth of hardware counters, but steep learning curve and environment specific.” |

**Key differences**:
- Go’s approach is **unified** – you don’t need `perf` (though it’s still useful for kernel‑level issues). The same `pprof` binary works on Windows, macOS, Linux, and even WASM.
- Java’s JFR can run continuously with <1% overhead, while Go’s CPU profiler (signal‑based) adds ~2–5% overhead – not suitable for 24/7 monitoring in most environments.
- Rust’s ecosystem is more fragmented: `criterion` for benchmarks, `pprof‑rs` for Go‑like profiles, `tokio-console` for async tracing. Go’s single toolchain is simpler for newcomers.

**Where Go shines**: Profiling a live production service with a single `curl` and viewing a flame graph in your browser without restarting the process or deploying agents.

---

## 5. Common Mistakes

### Mistake 1: Profiling with Incorrect Sampling Rates

```go
// BAD: Changing MemProfileRate after allocations have started
runtime.MemProfileRate = 1024 // too late – rate is fixed at init time
```

`MemProfileRate` must be set before the first allocation, ideally in `init()` or `main()` very early. Otherwise, the previously initialized allocator ignores the change.

**Fix**: Set at startup before any allocation-heavy init code runs.

### Mistake 2: Benchmark Dead‑Code Elimination

```go
// BAD: Compiler discards the allocation entirely
func BenchmarkWrong(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = make([]byte, 1024) // _ = ignore – result not used
    }
}
```

**Fix**: Assign to a package variable or call `testing.B.StopTimer()` + `runtime.KeepAlive`.

### Mistake 3: Profiling Too Short a Duration

A 5‑second CPU profile on a server with 1000 goroutines might miss infrequent but expensive operations. The default sampling rate is 100 Hz, so a 5‑second profile collects only 500 samples. If a critical function runs rarely, it may have zero samples.

**Fix**: Use at least 30 seconds for steady‑state services. For bursty workloads, use `curl` with `?seconds=120` or integrate continuous profiling (e.g., Pyroscope, Parca).

### Mistake 4: Misreading `inuse_space` vs `alloc_space`

```text
(pprof) top -cum
flat  flat%   sum%        cum   cum%
123MB 12.3% 12.3%      890MB 89.0%  encoding/json.(*decodeState).object
```

`flat` = memory allocated directly in this function. `cum` = inclusive – includes callees. A high `alloc_space` (cumulative) suggests repeated allocations, but `inuse_space` (live objects) might be low. Often, the real problem is allocation churn, not memory leak.

**Fix**: Use `-sample_index=inuse_space` to see what’s live; use `-sample_index=alloc_space` to see allocation hot spots.

### Mistake 5: Running `go test -bench` Without `-count`

Benchmarks are cached by default. If the code hasn’t changed, `go test` returns the previous result instantly – masking the effect of code changes.

**Fix**: `go test -bench=. -count=10` runs the benchmark 10 times and shows variance.

---

## 6. Performance Considerations

### Cost of Profiling Itself

| Profile type | Typical overhead | When to use |
|--------------|------------------|--------------|
| CPU (100 Hz) | 2–5% CPU, slight increase in tail latency | Development, staging, short production captures |
| Heap (`MemProfileRate=512KB`) | <1% CPU, no impact on latency | Always‑on in production (if sample rate is reasonable) |
| Mutex (`Fraction=5`) | 5–10% on high‑contention services | Staging only; too heavy for production |
| Block (`Rate=1000ns`) | 1–3% | Debugging channel/mutex blocking |
| Execution trace | 10–30% CPU, major latency spike | Short (seconds) debugging sessions only |

### Optimization Trade‑offs

**Reducing allocations** often leads to:
- **Higher CPU usage** (e.g., using `sync.Pool` adds atomic operations; reusing buffers may cause more cache misses)
- **More complex code** (object pooling, finalizers, or manual memory management via `unsafe`)

**Example**: A `sync.Pool` may reduce GC pressure but increase CPU due to pool contention. Profile both before and after.

**Algorithmic changes** (O(n²) → O(n log n)) dwarf micro‑optimizations. Always check for obvious algorithmic inefficiencies first – pprof’s flame graph will show you wide towers of self‑time.

**Inlining**: The Go compiler aggressively inlines small functions (≤ 80 nodes on the SSA representation). Inlined functions disappear from CPU profiles, making profiles “cleaner” but sometimes hiding costs. Use `-gcflags="-l"` to disable inlining during profiling if needed.

**Escape analysis**: A variable that escapes to heap (shown by `go build -gcflags="-m"`) can cause allocations. Profiling with `-memprofile` will highlight those allocation sites. Eliminating escapes (e.g., passing pointers instead of large structs by value) reduces GC pressure.

---

## 7. Best Practices

1. **Profile with real workloads**  
   Use production traffic (or a replay) when possible. Unit test benchmarks rarely reveal concurrency bottlenecks.

2. **Enable `net/http/pprof` behind an auth proxy**  
   ```go
   import _ "net/http/pprof"
   // Then expose /debug/pprof only to internal network or with basic auth
   ```

3. **Use `-benchmem` in every benchmark**  
   Allocation counts are as important as time.

4. **Compare profiles before/after changes**  
   ```bash
   go test -bench=. -cpuprofile=before.out
   # apply change
   go test -bench=. -cpuprofile=after.out
   go tool pprof -base before.out after.out
   ```
   The `-base` flag shows the difference between two profiles.

5. **Add user‑defined labels**  
   ```go
   pprof.Do(context.Background(), pprof.Labels("handler", "/v1/user"), func(ctx context.Context) {
       handleRequest(ctx)
   })
   ```
   Then filter profiles: `go tool pprof -tagignore='handler=/*'` or view tags in the web UI.

6. **Combine pprof with execution traces**  
   When a CPU profile shows high `runtime.futex` or `runtime.epollwait`, that’s blocked goroutines. Switch to `go tool trace` to see the scheduler view.

7. **Set a baseline**  
   Before any optimization, record a profile and a benchmark. Without a baseline, you cannot measure improvement or regression.

8. **Never optimize without profiling**  
   “Premature optimization is the root of all evil” – Knuth. Go’s compiler and runtime are sophisticated; your intuition is often wrong.

---

## 8. Examples

### Example 1: Finding a Memory Leak in a Long‑Running Service

```go
// leaky.go
package main

import (
    "log"
    "net/http"
    _ "net/http/pprof"
    "time"
)

var cache = map[string][]byte{}

func handler(w http.ResponseWriter, r *http.Request) {
    key := r.URL.Query().Get("key")
    // LEAK: never deletes from cache
    cache[key] = make([]byte, 10<<20) // 10MB
    w.Write([]byte("ok"))
}

func main() {
    go func() {
        log.Println(http.ListenAndServe(":6060", nil))
    }()
    http.HandleFunc("/store", handler)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

**Diagnosis**:

```bash
# After sending 10 requests with different keys:
curl http://localhost:6060/debug/pprof/heap > heap.pb.gz
go tool pprof -sample_index=inuse_space -top heap.pb.gz

# Output:
# (pprof) top
# 120MB of 120MB (100%) in main.handler
```

**Fix**: Add a TTL or LRU eviction, or use `runtime.SetFinalizer` with caution.

### Example 2: Benchmark + Optimization Workflow

```go
// parse_bench_test.go
package main

import (
    "strconv"
    "testing"
)

var globalInt int

func BenchmarkParseInt(b *testing.B) {
    data := "123456"
    var v int
    for i := 0; i < b.N; i++ {
        v, _ = strconv.Atoi(data)
    }
    globalInt = v
}
```

```bash
go test -bench=. -cpuprofile=cpu.out
go tool pprof -http=:8080 cpu.out
# Flame graph shows time spent in strconv.Atoi -> internal/bytealg
```

**Optimization**:

```go
// Precompute or use a faster parser if possible. 
// But in reality, Atoi is already fast. The lesson: 
// profile first, discover that this is NOT the bottleneck.
```

### Example 3: Trace Analysis for Goroutine Leak

```go
// trace_leak.go
package main

import (
    "os"
    "runtime/trace"
    "time"
)

func worker(ch <-chan int) {
    for range ch { // blocked forever if ch never closed
        // do work
    }
}

func main() {
    trace.Start(os.Stderr)
    defer trace.Stop()

    ch := make(chan int)
    for i := 0; i < 100; i++ {
        go worker(ch)
    }
    // Oops: never close ch, workers never exit
    time.Sleep(1 * time.Second)
}
```

```bash
go run trace_leak.go 2> trace.out
go tool trace trace.out
# Viewer → "Goroutines" → see 100 goroutines in "chan receive" state
```

**Fix**: Always close channels or use `context` for cancellation.

---

## 9. Summary & Exercises

### Summary

- **pprof** provides CPU, heap, mutex, and block profiling via the runtime – no external tools required.
- **Benchmarks** (`testing.B`) are the foundation of reproducible optimization, but you must prevent dead‑code elimination.
- **Execution traces** reveal the scheduler’s decisions, blocking, and GC interference.
- **Premature optimization is dangerous** – always profile before changing code.
- **Go’s philosophy**: self‑describing, low‑friction profiling that works identically on your laptop and in production.

### Exercises

**Exercise 1: Profile a naive Fibonacci implementation**  
Write a recursive Fibonacci function (`fib(n int) int`). Run a CPU profile while computing `fib(45)`. Identify the hot path. Then add memoization and profile again. Compute the speedup.

**Exercise 2: Detect and fix allocation churn in a JSON parser**  
Given a function that repeatedly unmarshals large JSON arrays from a `[]byte` using `json.Unmarshal`, use `go test -benchmem` to measure allocations. Refactor to use `json.Decoder` with `decoder.Token()` streaming API. Compare allocations per operation before/after.

**Exercise 3: Production‑ready pprof endpoint with security**  
Build an HTTP server that registers the default `net/http/pprof` handlers but requires an API key in the `Authorization` header. Use a middleware that checks the key before allowing access to `/debug/pprof/*`. Verify that a `curl` without the key receives 401.

**Exercise 4: Trace a blocked pipeline**  
Write a pipeline with three stages connected by buffered channels. Artificially slow down the second stage by adding a `time.Sleep`. Capture an execution trace and identify why the first stage occasionally blocks. Then increase the buffer size of the channel between stage1 and stage2, and compare traces.

**Exercise 5: Compare two sorting algorithms**  
Implement bubble sort and quicksort on `[]int` of length 10,000. Write benchmarks for both. Run `go test -bench=. -cpuprofile=sort.out`. Use `go tool pprof -diff_base=quicksort.out bubblesort.out` to generate a comparison. Which function consumes the most additional CPU?
