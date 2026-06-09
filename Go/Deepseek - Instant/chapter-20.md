## Chapter 20: Garbage Collection (GC)

Go’s garbage collector is a concurrent, tri-color, mark‑sweep collector optimized for low latency. Unlike runtime systems that pause the world for seconds, Go’s GC typically keeps stop‑the‑world pauses in the sub‑millisecond to low‑millisecond range. This chapter dissects how it works, why the Go team made specific trade‑offs, and how to write code that coexists peacefully with the collector.

---

### Basic Usage

You rarely interact with the GC directly, but the standard library exposes hooks for tuning and observation. The two most important packages are `runtime` and `runtime/debug`.

```go
package main

import (
    "log"
    "runtime"
    "runtime/debug"
    "time"
)

func main() {
    // 1. Force a GC cycle (blocking, stops the world briefly)
    runtime.GC()
    log.Println("manual GC triggered")

    // 2. Read memory statistics
    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    log.Printf("Heap allocated: %d bytes", m.HeapAlloc)

    // 3. Adjust GC target percentage
    //    default = 100 → GC runs when heap grows 100% since last cycle
    oldPercent := debug.SetGCPercent(200) // reduce GC frequency
    defer debug.SetGCPercent(oldPercent)

    // 4. Disable GC temporarily (extremely rare, usually a mistake)
    debug.SetGCPercent(-1)
    time.Sleep(5 * time.Second) // no GC during this period
    debug.SetGCPercent(oldPercent)

    // 5. Monitor GC via runtime metrics (Go 1.21+)
    log.Printf("GC cycle count: %d", m.NumGC)
}
```

**Observation with `GODEBUG=gctrace=1`**  
Set the environment variable before running your binary:

```bash
$ GODEBUG=gctrace=1 go run main.go
gc 1 @0.012s 2%: 0.026+0.49+0.017 ms clock, 0.21+0.28/0.38/0.58+0.13 ms cpu, 4->5->2 MB, 5 MB goal, 8 P
```

Each field tells a story: pause times, mark assist time, heap growth, and concurrency details. Learn to read this output; it’s the fastest way to diagnose GC issues.

---

### Under the Hood

Go’s GC is a **non‑generational concurrent tri‑color mark‑sweep** collector. Let’s unpack each term.

#### Tri‑Color Abstraction
The collector divides all heap objects into three colour sets:
- **White** – not yet scanned (candidates for garbage).
- **Grey** – reached but its children not yet scanned.
- **Black** – reached and fully scanned.

The algorithm:
1. Start with all objects white.
2. Mark roots (goroutine stacks, globals, etc.) grey.
3. While a grey object exists:  
   - Pick a grey object, scan its pointers.  
   - Pointed‑to white objects become grey.  
   - The scanned object becomes black.
4. When no grey objects remain, white objects are unreachable → sweep them.

#### Concurrent Execution Phases
Go splits the work into four phases to minimise stop‑the‑world (STW):

| Phase | STW? | Work done |
|-------|------|------------|
| **Mark Setup** | Yes (very short) | Enable write barrier, prepare background workers. |
| **Mark Concurrent** | No | Background goroutines scan grey objects; application runs simultaneously. |
| **Mark Termination** | Yes (short) | Complete marking, disable write barrier. |
| **Sweep Concurrent** | No | Free white objects; done lazily during allocation. |

**Write Barrier** – During concurrent marking, the mutator (your code) may change pointers (e.g., `a.next = b`). To preserve the tri‑colour invariant, Go uses a **dijkstra‑style insertion write barrier**. When you write `*slot = ptr`, the barrier shades `ptr` grey if `slot` is black. This prevents a black object from incorrectly pointing to a white object that later gets collected.

#### Pacing (GC Trigger)
Go does not use a fixed threshold. Instead, it employs a **pacing algorithm** that tries to keep the heap size within a target. The target is based on:
- `GOGC` (default 100) – when live heap grows by 100% since last GC, trigger new cycle.
- Current allocation rate – if the program allocates fast, the GC starts earlier.

The pacer adjusts the `trigger` dynamically. For example, after a GC that left 10 MiB live, with `GOGC=100`, the next GC triggers when heap reaches ~20 MiB. But if allocation is bursty, the pacer may start earlier to avoid overshoot.

#### No Generations, No Compaction
Go explicitly rejected generational and compacting collectors. Why?  
- **Generations** would require a write barrier for remembered sets and separate young/old heaps, increasing complexity and STW pauses for card tables.  
- **Compaction** (moving objects) would break Go’s unsafe pointers and the ability to take address of heap objects (`&Foo{}`). It would also require updating all pointers – a massive STW event.

Instead, Go relies on **concurrent sweeping** and the **pacer** to keep fragmentation manageable. Fragmentation is rarely severe because Go’s allocator (based on **size classes**) splits memory into fixed‑size spans. This reduces external fragmentation at the cost of internal fragmentation (wasted space within a span).

---

### Why This Design?

The Go team prioritised **low latency for network services** over raw throughput or memory efficiency. The design decisions reflect that philosophy.

**1. Concurrent over stop‑the‑world**  
Early Go (pre‑1.5) used a parallel stop‑the‑world collector. For web servers handling thousands of requests, a 100ms pause was catastrophic. The rewrite to a concurrent collector kept tail latency under control.

**2. No generational GC**  
Generational collectors excel when most objects die young. Go’s workloads (e.g., long‑lived goroutines, caches, connection pools) often have mixed object lifetimes. The complexity of a write barrier for the remembered set and the need for periodic full collections added little benefit. The Go team measured and decided: “Simplicity over hypothetical wins.”

**3. Non‑moving (no compaction)**  
Go exposes pointers to heap objects freely (e.g., `&user.Age`). Compaction would require finding and updating every pointer to a moved object – a global STW operation that nullifies concurrency gains. By keeping objects at fixed addresses, Go allows `unsafe.Pointer` and direct C interop (via `cgo`), albeit with responsibility.

**4. Pacing instead of user‑tuned knobs**  
In Java you tune `-Xmn`, `-XX:MaxTenuringThreshold`, etc. Go gives you `GOGC` and `GOMEMLIMIT` (Go 1.19+). The pacer automatically adapts to allocation patterns. This aligns with Go’s “less is more” – the average developer doesn’t need a GC PhD.

**The “Aha!” moment:** The GC is not your enemy; it’s a concurrent daemon that cooperates with your code via write barriers. The real performance problem is rarely the GC’s algorithm – it’s **allocation rate**. Reduce allocations, and the GC naturally becomes invisible.

---

### Competing Approaches

| Language | GC Strategy | Pause Trade‑offs | Go Comparison |
|----------|-------------|------------------|----------------|
| **Java (G1, ZGC)** | Generational, region‑based, concurrent (ZGC: sub‑ms pauses) | Low latency, but higher CPU overhead and complex tuning. | Go gives comparable latency with simpler configuration, but ZGC can beat Go on huge heaps (>100GB). |
| **C++** | Manual (no GC) | Zero pause, but risk of leaks, use‑after‑free, and double‑free. | Go trades absolute performance for safety. Use C++ when every microsecond matters and you can afford manual memory management. |
| **Rust** | Ownership + borrowing (compile‑time) | Zero runtime overhead, but steep learning curve. | Go’s GC imposes runtime cost; Rust is systems‑language alternative without pauses. |
| **Python (ref‑count + cycle GC)** | Reference counting (deterministic) + occasional mark‑sweep for cycles. | Reference counting overhead on every assignment; cycle detection pauses can be long. | Go’s concurrent GC avoids per‑assignment overhead and scales to many cores. |
| **C# (Workstation/Server GC)** | Generational, concurrent, optionally background. | Workstation GC prioritises low latency; server GC uses multiple heaps. | Both achieve similar goals, but Go’s pacer is simpler. C# offers finer control (e.g., `GC.Collect()` generations). |

**Key insight:** Go’s GC is not “the best” at any single metric. It’s the **best compromise** for building network services that must remain responsive under load without requiring a JVM tuning expert on your team.

---

### Common Mistakes

**1. Calling `runtime.GC()` manually**  
Forcing a GC cycle stops the world synchronously. The pacer already knows when to run. Manual calls usually make latency worse, not better.

```go
// WRONG: "helping" the GC
go func() {
    for range time.Tick(5 * time.Second) {
        runtime.GC() // Don't.
    }
}()
```

**2. Relying on `runtime.Finalizer`**  
Finalizers are unpredictable, delay garbage collection, and can resurrect objects. They run on a single goroutine and are not guaranteed to run at all before program exit.

```go
// FRAGILE: finalizer for cleanup
obj := &MyResource{}
runtime.SetFinalizer(obj, func(o *MyResource) {
    o.file.Close() // May never run, or run long after you needed it.
})
```

Use `defer` or explicit `Close()` methods instead.

**3. Keeping unnecessary pointers**  
A pointer to a small field inside a large struct prevents the whole struct from being collected. This is the **“spurious retention”** trap.

```go
type LargeBuffer [1 << 20]byte
type Wrapper struct {
    buf *LargeBuffer // If Wrapper lives, the 1MiB buffer lives
}
```

**4. Assuming `GOGC=off` (`debug.SetGCPercent(-1)`) is a performance win**  
Disabling GC saves CPU cycles but guarantees unbounded memory growth. Your container will OOM‑kill within minutes. Only valid for very short‑lived command‑line tools.

**5. Ignoring the write barrier cost**  
The insertion write barrier adds a few CPU instructions per pointer write. In extremely hot loops, this matters:

```go
// Each iteration incurs write barrier overhead
for i := range largeSlice {
    ptr := &someStruct
    largeSlice[i] = ptr // writes a pointer → write barrier
}
```

Where possible, batch modifications or use value types to avoid pointer writes.

---

### Performance Considerations

**GC Cost Model**  
Total GC cost ≈ (mark cost) + (sweep cost) + (write barrier cost). For a typical service:
- Mark cost is proportional to **live heap size** (not total heap).
- Sweep cost is proportional to **dead heap size** (freed objects).
- Write barrier cost is proportional to **pointer assignment frequency**.

**Metrics to Monitor**
- `GOGC=100` → GC runs when heap grows by 100% since last GC. Lower values (e.g., 50) reduce memory footprint but increase CPU usage. Higher values (e.g., 200) reduce CPU at the cost of memory.
- **Heap allocation rate** (bytes/sec) – this is the single most important number. If you allocate 1 GiB/s, the GC will spend 10–30% of CPU time.
- **Pause time percentiles** – use `histogram_pause_ns` from `runtime/metrics`.

**Escape Analysis Saves GC**  
Every heap allocation is a future GC burden. Prefer stack allocation when possible.

```go
// Escapes to heap (returned pointer)
func newPoint() *Point {
    return &Point{X: 1, Y: 2}
}

// Stays on stack (if not returned)
func usePoint() {
    p := Point{X: 1, Y: 2} // stack allocated
    // ...
}
```

**Big O & Scalability**  
- Mark work is O(live objects) – linear scan.
- The concurrent mark scales with number of CPU cores (parallel workers).
- The pacer ensures that GC CPU usage scales roughly with allocation rate. If you double allocation, GC CPU usage doubles (linear).

**Memory Limit (Go 1.19+)**  
`GOMEMLIMIT` caps total memory usage. When set, the GC becomes more aggressive before hitting the limit, preventing OOM kills.

```go
// Prefer environment variable
// export GOMEMLIMIT=8GiB

// Or programmatically
import "runtime/debug"
debug.SetMemoryLimit(8 << 30) // 8 GiB
```

Combined with `GOGC=off`, the memory limit alone drives GC decisions – useful for latency‑sensitive workloads where you want to trade CPU for memory stability.

---

### Best Practices

**1. Minimise heap allocations**  
- Use value receivers (`func (t T)`) instead of pointer receivers (`func (t *T)`) unless you need mutation or the struct is large (>128 bytes).
- Preallocate slices with `make([]T, 0, capacity)`.
- Use `strings.Builder` instead of `+` concatenation in loops.
- Avoid boxing via `interface{}` (now `any`) – each boxing allocates.

**2. Pool reusable objects**  
`sync.Pool` reduces allocation rate for short‑lived objects (e.g., request contexts, buffers).

```go
var bufferPool = sync.Pool{
    New: func() interface{} { return make([]byte, 0, 4096) },
}

func handleRequest() {
    buf := bufferPool.Get().([]byte)
    defer bufferPool.Put(buf[:0]) // Reset before putting back
    // use buf
}
```

**3. Let the pacer work**  
Do not tune `GOGC` unless you have measured. Start with the default, profile, then adjust. For memory‑constrained environments, set `GOMEMLIMIT` first.

**4. Use `runtime/metrics` for production monitoring**  
Go 1.21 introduced stable metric names. Sample them every 10s.

```go
import "runtime/metrics"

func sampleGC() {
    const metricName = "/gc/pauses:seconds"
    sample := []metrics.Sample{{Name: metricName}}
    metrics.Read(sample)
    // sample[0].Value.Float64() → total pause time since last read
}
```

**5. Write benchmarks with `testing.AllocsPerRun`**  
Always check allocation counts in performance‑sensitive code.

```go
func BenchmarkAllocs(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        // your code
    }
}
```

**6. Avoid finalizers**  
They are a code smell. Use explicit `Close()` and `defer` instead.

---

### Examples

**Example 1: Observing GC with `runtime.MemStats`**

```go
package main

import (
    "fmt"
    "runtime"
    "time"
)

func allocate() {
    _ = make([]byte, 10<<20) // 10 MiB
}

func main() {
    var mem runtime.MemStats
    runtime.ReadMemStats(&mem)
    fmt.Printf("Initial heap: %d MB\n", mem.HeapAlloc/1024/1024)

    allocate()
    runtime.GC()
    runtime.ReadMemStats(&mem)
    fmt.Printf("After alloc+GC: %d MB\n", mem.HeapAlloc/1024/1024)

    // Let GC run concurrently
    go func() {
        for range time.Tick(2 * time.Second) {
            runtime.ReadMemStats(&mem)
            fmt.Printf("Live heap: %d MB, GC cycles: %d\n",
                mem.HeapAlloc/1024/1024, mem.NumGC)
        }
    }()
    time.Sleep(10 * time.Second)
}
```

**Example 2: GC‑friendly vs. unfriendly code**

```go
// BAD: Many small allocations
func sumOfSquaresBad(n int) int {
    var squares []int // nil slice, will reallocate many times
    for i := 0; i < n; i++ {
        squares = append(squares, i*i) // may allocate each append
    }
    sum := 0
    for _, v := range squares {
        sum += v
    }
    return sum
}

// GOOD: Preallocate and reuse stack where possible
func sumOfSquaresGood(n int) int {
    squares := make([]int, 0, n) // one allocation
    for i := 0; i < n; i++ {
        squares = append(squares, i*i)
    }
    sum := 0
    for _, v := range squares {
        sum += v
    }
    return sum
}

// Benchmark results (typical):
// Bad:  1000 allocs/op,  800 ns/op
// Good: 1 alloc/op,       200 ns/op
```

**Example 3: Using `sync.Pool` with a realistic HTTP handler**

```go
type RequestContext struct {
    UserID int
    Data   []byte
}

var ctxPool = sync.Pool{
    New: func() any { return &RequestContext{Data: make([]byte, 0, 1024)} },
}

func handler(w http.ResponseWriter, r *http.Request) {
    ctx := ctxPool.Get().(*RequestContext)
    defer ctxPool.Put(ctx)

    ctx.UserID = 42
    ctx.Data = ctx.Data[:0] // reset length, keep capacity
    // process request...
    w.WriteHeader(http.StatusOK)
}
```

---

### Summary & Exercises

**Summary**  
- Go’s GC is concurrent, tri‑color mark‑sweep, optimised for low latency at the cost of higher CPU usage.
- It avoids generations and compaction to keep pointers stable and implementation simple.
- The pacer (`GOGC` and `GOMEMLIMIT`) dynamically adjusts GC frequency.
- Performance is dominated by **allocation rate** – reduce allocations via pooling, preallocation, and stack allocation.
- Tools: `GODEBUG=gctrace=1`, `runtime/metrics`, `go test -bench=. -benchmem`.

**Key takeaways for the seasoned engineer**  
- Do not manually call `runtime.GC()`.
- Treat finalizers as dangerous; use explicit cleanup.
- Prefer value receivers for small structs to reduce heap pressure.
- Always measure before tuning `GOGC` or `GOMEMLIMIT`.

---

### Exercises

**Exercise 1: Build a GC‑aware cache**  
Implement a time‑based cache (`TTLCache`) that stores values with an expiry timestamp. The cache must:
- Automatically evict expired entries on reads.
- Run a background goroutine every 10 seconds to sweep expired entries.
- Use `sync.Pool` to reuse expired entry structs and reduce allocations.
- Measure the allocation rate of `Get` and `Set` operations with `testing.Benchmark` and `-benchmem`.

**Exercise 2: Diagnose and fix a GC‑heavy service**  
Given the following faulty snippet, identify why it causes high GC CPU usage. Rewrite it to reduce allocations by 90%.

```go
func processLogs(logs []string) map[string]int {
    result := make(map[string]int)
    for _, line := range logs {
        parts := strings.Split(line, ",")
        key := parts[0] + ":" + parts[1] // allocates new string
        result[key]++
    }
    return result
}
```

**Exercise 3: Tune `GOMEMLIMIT` for a realistic workload**  
Write a program that allocates 100 MiB/s continuously (simulate streaming data). Run it with:
- No limit (default) – observe memory growth and OOM risk.
- `GOMEMLIMIT=50MiB` – observe how the GC becomes aggressive.
- `GOMEMLIMIT=200MiB` with `GOGC=200` – measure the trade‑off between CPU usage and memory footprint using `time` and `runtime.ReadMemStats`.  
Explain which configuration minimises tail latency for a latency‑sensitive proxy.
