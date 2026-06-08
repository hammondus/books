## Chapter 20: Garbage Collection (GC)

Go is a garbage‑collected language. It frees you from manual memory management, but it does not free you from *thinking* about memory. For a seasoned engineer coming from C++ or Rust, the runtime’s collector can feel like a black box that occasionally stabs you in the tail latency. For someone used to the JVM or .NET, Go’s approach may seem deceptively simple. This chapter peels back the layers: how the collector works, why it looks the way it does, and how to write code that stays friendly to it—while keeping your service’s latency predictable and your memory bill low.

---

### 1. Basic Usage

In day‑to‑day Go, you rarely interact with the GC directly. Allocation happens anywhere you use `new`, `make`, take the address of a local variable, or create a composite literal. The collector runs in the background, mostly concurrently. However, Go exposes a handful of knobs for when you need to take control.

**Inspecting GC activity**
The `runtime` package provides `ReadMemStats`, which gives a snapshot of heap size, number of GC cycles, and pause times.

```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

func main() {
	var m runtime.MemStats
	for i := 0; i < 5; i++ {
		// Allocate 1 MB
		_ = make([]byte, 1<<20)
		runtime.ReadMemStats(&m)
		fmt.Printf("HeapAlloc: %d KB, NumGC: %d, PauseTotalNs: %d\n",
			m.HeapAlloc/1024, m.NumGC, m.PauseTotalNs)
		time.Sleep(100 * time.Millisecond)
	}
}
```

**Tuning GC frequency**
The most important knob is **GOGC** (default 100). It controls the percentage of new heap data that triggers a collection. A value of 100 means the heap may double in size between GC cycles. The environment variable `GOGC` or the function `debug.SetGCPercent` adjusts it at runtime.

```go
import "runtime/debug"

func main() {
	debug.SetGCPercent(200) // allow heap to triple before next GC
	// ... your workload ...
}
```

Set `GOGC=off` to disable automatic collections entirely (useful for short‑lived batch jobs that can just let the OS reclaim memory on exit).

**Soft memory limit (Go 1.19+)**
For services running inside containers with hard memory limits, `GOMEMLIMIT` (or `debug.SetMemoryLimit`) tells the runtime to keep the live heap under a specific byte limit. The GC will be triggered more aggressively when approaching that ceiling.

```go
debug.SetMemoryLimit(1 << 30) // 1 GiB
```

**Manual GC invocation**
`runtime.GC()` forces a synchronous collection. It blocks until sweeping completes. Use it sparingly—it can devastate throughput. It’s acceptable in tests or right before a memory‑heavy phase to reset the heap.

```go
runtime.GC()
```

These controls are the steering wheel, not the engine. To drive well, you must understand what happens when you turn it.

---

### 2. Under the Hood

Go’s GC is a **concurrent, tricolor mark‑and‑sweep collector**. It is not generational, not moving, and not reference‑counted. The design optimises for **low pause times** while accepting a moderate throughput overhead.

#### The Phases of a GC Cycle

A single GC cycle consists of several overlapping phases:

1. **Sweep termination** – Finish any leftover sweeping from the previous cycle so the heap is ready.
2. **Mark setup** – Enable the write barrier, stop the world (STW) briefly to prepare all goroutines for concurrent marking.
3. **Marking** – The bulk of the work. The world is running, and the collector finds all reachable objects. Goroutines that allocate during marking assist the collector to avoid falling behind (the “assist” mechanism).
4. **Mark termination** – A short STW to finish marking (e.g., rescan roots, drain remaining work queues) and turn off the write barrier.
5. **Sweeping** – Reclaiming memory from unreachable objects. Sweeping happens **concurrently**, interleaved with allocation: when a goroutine needs a new span, it may sweep the next free span before using it. This avoids a separate STW sweeping phase.

#### Tricolor Abstraction

Objects move through three colors:

- **White** – Not yet visited. Initially all objects are white. At the end of marking, white objects are unreachable and become garbage.
- **Grey** – Visited, but its pointers may still point to white objects. The work queue.
- **Black** – Visited, and all its outgoing pointers have been followed. No white children remain.

The invariant is: **no black object may point to a white object**. To maintain this during concurrent mutation, Go uses a **write barrier** (insert barrier) that shades (greys) the object being pointed to before the pointer is written. This ensures the collector never loses track of reachable objects that get moved or newly referenced during marking.

#### Stack Scanning

Goroutine stacks are scanned conservatively. The runtime scans all running goroutine stacks for potential pointers to the heap. When a goroutine is preempted, its stack is considered a root. This means that even local variables that *look* like pointers can keep objects alive—a subtle point we’ll revisit in “Common Mistakes.”

#### The GC Pacer

The **pacer** decides when to start the next GC cycle. It uses a feedback loop based on the current heap size, the target heap size (derived from GOGC and live heap), and the allocation rate. If allocation is fast, the collector will start sooner to prevent the heap from overshooting the target. If the CPU is already saturated with marking assists, the pacer may let the heap grow a bit larger to reduce assist overhead. The pacer is the key reason you rarely need to tune GOGC beyond the default.

#### Memory Layout and Allocator

Go’s memory allocator is based on **TCmalloc**. The heap is divided into **spans** (pages) and **size classes**. Small objects (≤32 kB) are allocated from per‑P caches to avoid contention. The collector sweeps spans page‑by‑page, returning unused pages to the OS only when the entire span is free—which is why a fragmented heap can appear larger than live memory.

---

### 3. Why This Design?

Go’s GC philosophy is a deliberate trade‑off, shaped by the kind of software Google writes: long‑running servers that serve thousands of concurrent requests. For these workloads, **tail latency matters more than peak throughput**. A 100‑ms stop‑the‑world pause is unacceptable when your 99th percentile latency is 50 ms.

**Why not a generational GC?**
Generational collectors assume the “weak generational hypothesis” (most objects die young) and segregate the heap into young and old generations. They achieve high throughput by collecting the nursery frequently and cheaply. However, they also introduce complexity: write barriers between generations, remembered sets, and often require moving objects to combat fragmentation. Go’s designers judged that a concurrent, non‑generational collector can achieve low pause times with far less complexity. Moreover, Go’s escape analysis already moves many short‑lived objects to the stack, reducing the benefit of a nursery.

**Why not reference counting?**
Reference counting would eliminate global pauses but imposes per‑assignment overhead, cannot handle cycles without a supplementary cycle detector, and is generally slower for pointer‑heavy graphs. The team opted for a mark‑sweep approach to avoid pervasive atomic operations on every pointer write.

**Why not a moving (compacting) collector?**
Compacting GC reduces fragmentation and simplifies allocation, but moving objects requires updating every pointer that refers to them—a tricky task in a language that exposes pointers directly. Go values the ability to take the address of a struct field and pass it around without the runtime rewriting it. A non‑moving collector keeps the programming model simple.

**Simplicity and predictability**
The entire GC implementation is a single, well‑factored piece of the runtime. It has no pluggable algorithms, no ergonomic tuning “options” beyond GOGC and GOMEMLIMIT. The Go team believes that predictability and low cognitive load for the developer are worth the throughput cost. You can reason about GC behaviour without being a VM expert.

---

### 4. Competing Approaches

Every language with automatic memory management makes a different choice. Let’s compare Go to the major ecosystems you already know.

#### Java (HotSpot JVM)

Java offers multiple garbage collectors tuned for different goals: Parallel (throughput), G1 (balanced), ZGC and Shenandoah (sub‑ms pauses). These are generational, compacting collectors with years of heuristics. Java’s GCs can achieve higher throughput than Go’s at the cost of significant complexity and tuning. ZGC, for example, keeps pauses under 1 ms even on multi‑terabyte heaps, but it relies on multi‑mapping and coloured pointers that require hardware support and non‑trivial tuning. Java prioritises **maximum throughput with configurable pause targets**; Go prioritises **simplicity with good‑enough latency by default**.

#### .NET (C#)

The .NET GC has both workstation and server modes, with generational, compacting, and concurrent background marking. It also supports pinned objects and large object heaps. .NET’s GC is more feature‑rich but also more configurable than Go’s. It typically outperforms Go on raw allocation throughput, but again demands deeper knowledge to tune well.

#### Python (CPython)

Python uses reference counting as its primary memory management, supplemented by a cycle detector for garbage objects. Reference counting is deterministic (objects are freed as soon as their reference count drops to zero), which avoids pauses but introduces overhead on every pointer manipulation. The GIL protects reference count updates, making concurrent memory management a non‑issue—and also a limitation. Python’s approach shows why Go avoided reference counting: performance at scale is poor for pointer‑rich workloads.

#### Rust

Rust has no GC. Its ownership model and borrow checker enforce memory safety at compile time, eliminating the need for a runtime collector. This gives Rust maximum performance and predictable latency, but at a steep learning curve. The trade‑off is explicit memory management with help from the compiler versus Go’s convenience and faster development cycles. For systems where every microsecond counts, Rust’s approach is superior; for most network services, Go’s GC is a pragmatic sweet spot.

#### C++

C++ offers manual memory management (`new`/`delete`), smart pointers (reference counting), and optionally a Boehm‑style conservative GC. The cognitive overhead of keeping memory safe is entirely on the developer. Go’s GC is one of the reasons teams migrate from C++ to Go for non‑kernel, non‑real‑time applications: it removes an entire class of bugs.

**Bottom line:** Go occupies a unique niche—a language with a simple, automatic GC that delivers predictable, low latency without requiring you to become a GC tuning wizard. It sacrifices maximum throughput for operational simplicity, and for the majority of server‑side software that is the right trade.

---

### 5. Common Mistakes

**1. Ignoring GOGC in containerised environments**
A container with a 256 MiB memory limit may run a Go service that defaults to GOGC=100. If the live heap is 100 MiB, the runtime will allow the heap to grow to ~200 MiB before the next GC, which might be dangerously close to the limit. Without `GOMEMLIMIT`, the process can be OOM‑killed before the GC has a chance to reclaim. Set both `GOMEMLIMIT` and `GOGC` appropriately.

**2. Retaining pointers in long‑lived maps**
A map holding pointers to otherwise unreachable objects prevents the GC from collecting them.

```go
cache := make(map[string]*bigStruct)
// later: instead of deleting, you only remove the key from your logic?
// The pointer remains in the map, so the bigStruct stays alive forever.
```

Always delete map entries when they are no longer needed, or use a weakly‑referenced cache (with channels, timers, or third‑party libraries) if you need eviction.

**3. Over‑using pointers**
A struct field that is a pointer forces the entire struct to be allocated on the heap if it escapes. Value types can often stay on the stack, reducing GC pressure. Favour embedding values rather than pointers when the lifetime is bounded and the struct is small.

**4. Misusing finalizers**
`runtime.SetFinalizer` attaches a function that runs when the GC discovers an unreachable object. It seems like a destructor, but:

- Finalizers run in a dedicated goroutine, not synchronously.
- They may delay memory reclamation because the object isn’t freed until the finalizer returns.
- The order is not guaranteed.
- They can resurrect objects if they store them again.

Use finalizers only for niche resource‑reclamation (e.g., closing a file descriptor in a data structure that users might forget to close), and always provide an explicit `Close` method.

**5. Assuming GC is free**
Every heap allocation incurs CPU overhead: the write barrier, assist time, and sweeping. In a hot loop, allocating a `[]byte` instead of reusing a buffer adds significant GC pressure. A service doing 100k allocations/second can easily spend 10‑20% of CPU time in GC‑related work. Profile, not guess.

**6. Leaking goroutines**
A goroutine blocked on a channel that will never receive keeps its stack and any heap objects it references alive. GC cannot collect what is still reachable. Always ensure goroutines can exit.

---

### 6. Performance Considerations

Go’s GC cost is largely proportional to **two things**: the amount of live heap (scan work) and the **allocation rate** (pace of GC cycles). Understanding this lets you predict the impact.

- **Allocation count**: Each allocation that escapes to the heap adds to the “next GC target” calculation. Tools like `testing.B` report `allocs/op` and `bytes/op`. Reducing these numbers directly lowers GC CPU time.
- **Live heap size**: A larger live heap means more roots (global variables, goroutine stacks) and more objects to scan. The mark phase must traverse all live objects, so keeping data structures lean has a double benefit: less memory and faster collections.
- **GOGC tuning**: Raising GOGC reduces GC frequency but increases peak memory. Lowering GOGC does the opposite. For a service that rarely exceeds 200 MiB live heap but has 4 GiB available, you can set GOGC=500 to trade memory for throughput.
- **GOMEMLIMIT**: Sets a hard ceiling. The pacer will request GC cycles much more aggressively when approaching the limit, potentially causing higher assist overhead. This is better than an OOM kill, but you should still provide enough headroom.
- **CPU overhead of assists**: If the mutator allocates faster than the background mark worker can scan, the allocator goroutine must “assist” (do marking work itself). This can increase request latency. Monitoring `GOMAXPROCS` utilisation and assist time (`runtime.ReadMemStats` field `GCCPUFraction`) reveals whether assists are a problem.
- **Write barrier cost**: Every pointer update in the heap incurs a write‑barrier check during concurrent marking. This cost is small but pervasive. Data‑structure designs that minimise pointer mutations during GC cycles (e.g., building new structures and swapping atomically) can help in extreme cases.
- **Stack vs. heap**: The compiler’s escape analysis decides whether a variable can live on the stack. A value on the stack is zero‑cost for GC—it disappears when the function returns. Encourage stack allocation by:
  - Keeping data small and passing it by value.
  - Avoiding returning pointers to local variables (unless you explicitly want it to escape).
  - Using `go build -gcflags="-m"` to see escape analysis decisions.

The bottom line: treat allocation like a CPU expense. Profile to find allocation hotspots, then refactor.

---

### 7. Best Practices

**1. Reduce allocations first**
Before tuning GC knobs, make your code allocate less. Common tactics:

- Reuse buffers via `sync.Pool`.
- Pre‑allocate slices with `make([]T, 0, expectedSize)`.
- Use strings.Builder instead of `+` concatenation in loops.
- Avoid `time.Tick` (it leaks), use `time.NewTicker` and stop it.
- Pass by value when the struct is small and its lifetime is function‑scoped.
- Keep maps small, or use slices of key‑value pairs for small collections.

**2. Set reasonable GOGC and GOMEMLIMIT**
For production services, start with GOGC=100 and set `GOMEMLIMIT` to something like 80% of the container memory limit. Monitor memory and GC metrics, then adjust. Never set `GOGC=off` in a long‑running service unless you have an explicit heap‑management strategy (and strong nerves).

**3. Profile before tuning**
Use `pprof` with `-alloc_objects` and `-inuse_objects` to locate the source of allocations. Analyse GC pauses via the `/debug/pprof/trace?seconds=30` endpoint. The trace viewer shows exactly when STW events occur and how long they last. Optimise the code, not the flags.

**4. Use `sync.Pool` for temporary objects**
`sync.Pool` keeps a set of reusable objects that are automatically cleared by the GC when no longer referenced. It works best for short‑lived objects created in tight loops.

```go
var bufPool = sync.Pool{
	New: func() any { return make([]byte, 0, 1024) },
}

buf := bufPool.Get().([]byte)
defer bufPool.Put(buf) // return it for reuse
```

**5. Avoid finalizers unless absolutely necessary**
Prefer explicit cleanup functions (`Close`, `Cleanup`) and rely on `defer` or context cancellation. If you must use a finalizer, ensure the object is not resurrected and that the finalizer quickly returns.

**6. Let Go manage the heap**
Don’t manually `free` or try to outsmart the GC with custom allocators (unless you have a proven, profiled reason). The runtime’s allocator and GC are tightly integrated; fighting them usually backfires.

**7. Design for GC friendliness**
- Prefer slices of values over slices of pointers when the elements are small.
- Break large monolithic structures into smaller, independently collectible pieces.
- Use channels or `sync.Cond` for notification instead of holding pointers in a collection forever.

---

### 8. Examples

**Example 1: Measuring GC impact on a heavy allocator**

```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

func main() {
	// Start with a known state
	runtime.GC()
	var m1, m2 runtime.MemStats
	runtime.ReadMemStats(&m1)

	// Simulate a phase that allocates a lot
	for i := 0; i < 1_000_000; i++ {
		_ = make([]byte, 100) // each escapes to heap
	}
	runtime.ReadMemStats(&m2)

	fmt.Printf("Alloc: %v B, TotalAlloc: %v B\n", m2.HeapAlloc-m1.HeapAlloc, m2.TotalAlloc-m1.TotalAlloc)
	fmt.Printf("NumGC: %d, PauseNs: %d\n", m2.NumGC-m1.NumGC, m2.PauseTotalNs-m1.PauseTotalNs)
	time.Sleep(time.Second) // let console catch up
}
```

**Example 2: Using GOGC to trade memory for throughput**

```go
package main

import (
	"fmt"
	"os"
	"runtime"
	"runtime/debug"
	"time"
)

func main() {
	gogc := "100"
	if len(os.Args) > 1 {
		gogc = os.Args[1]
		debug.SetGCPercent(mustAtoi(gogc))
	}
	fmt.Printf("GOGC=%s\n", gogc)

	start := time.Now()
	for i := 0; i < 10_000_000; i++ {
		_ = make([]byte, 16) // small allocation, but many
	}
	fmt.Printf("Duration: %v\n", time.Since(start))
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	fmt.Printf("Max heap: %d MB\n", m.HeapSys/1024/1024)
}

func mustAtoi(s string) int {
	n, _ := fmt.Sscanf(s, "%d", new(int))
	return n
}
```

Run with `go run main.go 100` vs `go run main.go 500` – the higher GOGC value will show a larger maximum heap but likely a faster total time due to fewer GC cycles.

**Example 3: Memory leak via forgotten map key**

```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

type data struct {
	payload [1024]byte
}

func main() {
	cache := make(map[string]*data)
	for i := 0; i < 1000; i++ {
		key := fmt.Sprintf("key%d", i)
		cache[key] = &data{}
		// intention: keep only last 100 entries
		if len(cache) > 100 {
			// BUG: should delete the oldest key, but this does nothing
			// because range order is unpredictable.
			for k := range cache {
				delete(cache, k)
				break // deletes a random key
			}
		}
	}
	runtime.GC()
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	fmt.Printf("HeapAlloc: %d KB\n", m.HeapAlloc/1024)
	time.Sleep(time.Second)
}
```

The map still holds all 1000 pointers, wasting memory.

---

### 9. Summary & Exercises

**Summary**

Go’s concurrent, tricolor mark‑and‑sweep GC is a cornerstone of the language’s simplicity and suitability for servers. It delivers low‑latency collections with zero developer effort, yet leaves you with measurable control through GOGC and GOMEMLIMIT. The key to high‑performance Go is not outsmarting the collector, but feeding it less work: reduce allocations, favour the stack, and keep your live heap small. Monitor GC behaviour, profile before tuning, and always design with the “share memory by communicating” philosophy—it keeps object graphs simpler and friendlier to the GC.

**Exercises**

1. **Build a size‑bounded cache with GC‑aware eviction**
   Write a concurrent cache (e.g., `map[string]any` protected by a `sync.RWMutex`) that respects a maximum memory footprint. Use `runtime.ReadMemStats` or the `GOMEMLIMIT` mechanism to trigger eviction when the heap exceeds a threshold. The cache must support concurrent readers/writers and evict the least‑recently‑used entries when memory is tight. Bonus: use a background goroutine that periodically checks memory and evicts entries, ensuring the service never OOMs.

2. **Instrument a production‑like HTTP server for GC pauses**
   Extend an `http.Handler` to record GC pause duration and frequency using `runtime.ReadMemStats`. Expose these via a `/debug/gc` endpoint that returns JSON with percentiles (p50, p99) over the last minute. Use this instrumentation to experiment with different GOGC settings and observe the trade‑off between response latency and memory usage under a load generator (e.g., `wrk`).

3. **Refactor an allocation‑heavy function to be GC‑friendly**
   Take a function that constructs a complex data structure (e.g., a JSON marshaler that builds many intermediate maps) and refactor it to minimise allocations. Use `go test -bench . -benchmem` to measure `allocs/op` before and after. Try strategies: `sync.Pool`, pre‑allocated slices, switching from `map[string]interface{}` to a struct with fields where possible. Document the allocation count improvement and its effect on tail latency in a benchmark that runs with `GOMAXPROCS=1` to emphasise GC assist impact.
