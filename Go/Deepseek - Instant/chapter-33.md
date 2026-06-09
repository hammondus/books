# Chapter 33: Runtime & Memory Model

## 1. Basic Usage

The Go memory model defines **happens‑before** relationships that establish visibility guarantees across goroutines. You never interact with the memory model directly—instead, you use synchronization primitives that *imply* these relationships.

Here’s a correct pattern where a channel guarantees that the write to `data` is visible before the read:

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    var data int
    done := make(chan struct{})

    go func() {
        data = 42                     // write A
        close(done)                   // write B (send on channel)
    }()

    <-done                            // read C (receive from channel)
    fmt.Println(data)                 // read D – guaranteed to see 42
}
```

A mutex establishes happens‑before as well:

```go
var (
    mu   sync.Mutex
    data int
)

func write() {
    mu.Lock()   // lock L1
    data = 42   // write A
    mu.Unlock() // unlock U1
}

func read() int {
    mu.Lock()   // lock L2
    v := data   // read B
    mu.Unlock()
    return v
}
```

If `write()` completes before `read()` starts (in real time), the unlock of `mu` in `write` happens‑before the lock of `mu` in `read`, so `read` sees `42`.

Atomic operations also provide ordering guarantees. The following uses `sync/atomic` to implement a **try‑lock** pattern:

```go
var flag int32

func tryLock() bool {
    return atomic.CompareAndSwapInt32(&flag, 0, 1)
}

func unlock() {
    atomic.StoreInt32(&flag, 0)
}
```

`CompareAndSwap` establishes a happens‑before relationship for the load and store – changes made before a successful CAS are visible to the goroutine that succeeds.

## 2. Under the Hood

### Happens‑Before Formalised

The Go memory model is defined in a document (updated for Go 1.19+). The core rule: within a single goroutine, the program order is a happens‑before order. Across goroutines, specific synchronization operations create edges:

- **Channel send** `ch <- v` happens‑before the corresponding receive `<-ch` that completes.
- **Close of a channel** happens‑before a receive that returns zero value (because the channel is closed).
- **Unlock** on a mutex happens‑before any subsequent **Lock** of that mutex.
- **`sync.Once`** : the first call to `f()` in `once.Do(f)` happens‑before any call to `once.Do` returns.
- **`sync/atomic`** : a `Store` happens‑before any `Load` that observes that store (subject to the same address and alignment).
- **`sync.WaitGroup`** : a `wg.Add` that increments from 0 to N happens‑before a `wg.Wait` that observes the counter becoming 0.
- **Goroutine creation** `go f()` happens‑before the start of `f()`.
- **Goroutine exit** is *not* guaranteed to happen‑before anything – you must explicitly synchronise.

### Compiler and CPU Reordering

Modern CPUs and compilers reorder memory operations to improve performance. Without constraints, you could observe a write after a read that appears later in source code. The Go compiler and runtime insert **memory barriers** at every synchronization point:

- **Store barrier** (after a `Store` / `Unlock` / channel send) forces preceding writes to become visible.
- **Load barrier** (before a `Load` / `Lock` / channel receive) forces subsequent reads to see writes that happened before the barrier.

The exact barrier instructions differ per architecture (e.g., `DMB` on ARM64, `LOCK` prefix on x86). Go abstracts these away – you only need to use the documented primitives.

### Stack Growth & Scheduling

Each goroutine starts with a small stack (≈2KB). When the stack overflows, the runtime copies it to a larger block. This stack copy requires **stack barriers** (older Go versions) or a concurrent stack copying algorithm (Go 1.14+). Importantly, a stack growth does *not* create a happens‑before edge between goroutines – it’s an internal implementation detail.

The scheduler preempts goroutines at **safe points** (function calls, loops with explicit preemption checks). These preemption points *may* act as scheduling fences: when a goroutine is parked (e.g., blocking on a channel), all its prior writes are flushed to memory before another goroutine is unparked. But you must never rely on that – only documented synchronization is guaranteed.

## 3. Why This Design?

Go’s memory model is deliberately **stronger than C++’s** but **weaker than sequential consistency**. The design goals:

1. **Make correct code the obvious code** – Programmers should not have to reason about relaxed atomics or acquire/release semantics. Use channels or mutexes, and the memory model “just works”.
2. **Support lock‑free programming when necessary** – The `sync/atomic` package provides compare‑and‑swap and load/store with ordering guarantees (the default is *sequential consistency* for atomic operations, unlike C++ which defaults to relaxed).
3. **Avoid undefined behaviour** – In C++, a data race (unsynchronised concurrent read/write) is undefined – it can format your hard drive. In Go, a data race is *defined* as producing some value, but the result is nondeterministic and the race detector will flag it. Go does not let the compiler assume away races.
4. **Encourage communication** – The strongest happens‑before edges are associated with channels, reinforcing the “share memory by communicating” pillar.

The Go team chose **Sequential Consistency for Data‑Race‑Free (SC‑DRF)** programs – if your program has no data races, its behaviour is sequentially consistent (i.e., there exists some interleaving of operations that respects program order and the outcome is deterministic). This matches Java’s memory model (since JSR‑133) and is much easier to reason about than C++’s fine‑grained control.

## 4. Competing Approaches

| Language | Memory Model | Default Atomics | Data Race Behaviour |
|----------|--------------|----------------|---------------------|
| **Go** | SC‑DRF + documented sync ops | SeqCst (strong) | Defined but nondeterministic; race detector available |
| **Java** | SC‑DRF + `volatile` (seqcst) | `VarHandle` with ordering modes | “Benign” races allowed but risky |
| **C++** | Weak – six ordering modes (relaxed, acquire, release, acq_rel, seqcst, consume) | Relaxed by default | **Undefined behaviour** – nasal demons |
| **Rust** | Same as C++ (via `std::sync::atomic`) | Relaxed by default | **Undefined behaviour**, but ownership prevents many races |
| **Zig** | Exposes compiler/CPU barriers; no language‑level model | None built‑in; use `@atomic` | Unspecified; relies on LLVM |

**Why Go differs**:  
- **C++** gives maximum performance for lock‑free data structures (e.g., RCU, hazard pointers) but at the cost of enormous complexity. Go prioritises simplicity over nanosecond optimisations.  
- **Java** is similar to Go (SC‑DRF) but `volatile` has different semantics (it also prevents reordering with non‑volatile accesses). Go’s `sync/atomic` provides a clear mental model: atomic ops are sequentially consistent.  
- **Rust** uses the C++ model but eliminates many races at compile time through ownership. Go lacks that, so it compensates with a stronger default and a runtime race detector.

## 5. Common Mistakes

### 1. Double‑Checked Locking Without Atomic

```go
var once bool
var mu sync.Mutex
var resource *Resource

func getResource() *Resource {
    if once == false {          // read A (not atomic)
        mu.Lock()
        if once == false {      // read B (under lock)
            resource = new(Resource)
            once = true         // write C
        }
        mu.Unlock()
    }
    return resource
}
```

This is **broken** in Go because the first read `once == false` may see an outdated value (stale cache) and return `nil` or partially initialised `resource`. The memory model requires a happens‑before edge between `write C` and any read that observes `true`. Use `sync.Once` or `atomic.Bool`.

### 2. Assuming Goroutine Exit Synchronises

```go
var data int

func main() {
    go func() {
        data = 42
        // goroutine exits – NO implicit happens‑before
    }()
    time.Sleep(time.Millisecond) // wrong!
    fmt.Println(data)
}
```

The sleep does **not** create a happens‑before edge. The program may print `0` or `42`. Always use a channel, `sync.WaitGroup`, or mutex.

### 3. Misusing `sync/atomic` for Complex Invariants

```go
var counter int64

func increment() {
    atomic.AddInt64(&counter, 1)
}
```

This is fine for a counter. But if you have two dependent variables, atomic operations on each individually do **not** create a transaction:

```go
var x, y int64

func setXY(a, b int64) {
    atomic.StoreInt64(&x, a) // write A
    atomic.StoreInt64(&y, b) // write B – no happens‑before between A and B across goroutines
}
```

Another goroutine may see `x` updated but not `y`. Use a mutex to protect invariants.

### 4. Finalizers Are Not for Synchronisation

`runtime.SetFinalizer` is **not** guaranteed to run promptly or at all. You cannot rely on a finalizer to release a lock or signal that an object is unused.

## 6. Performance Considerations

### Cost of Memory Barriers

- **Store barrier** (e.g., `Unlock`, channel send) costs ≈10–50 ns on modern CPUs, depending on contention and cache coherence.
- **Load barrier** (e.g., `Lock`, channel receive) similar cost.
- **Atomic Compare‑And‑Swap** may take 100–200 ns under contention because it forces a cache line to be owned exclusively.

### False Sharing

When two goroutines frequently update different fields that reside on the same cache line (typically 64 bytes), the CPU constantly invalidates and reloads that line. This can be worse than using a mutex. **Pad** your structs or reorganise data:

```go
type Padded struct {
    a uint64
    _ [56]byte // pad to 64 bytes
    b uint64
}
```

### GC Interaction

The garbage collector’s **write barrier** (needed for concurrent marking) adds a small overhead to every pointer store. This is orthogonal to the memory model but can be mistaken for a synchronisation barrier – it is **not** a happens‑before edge.

### Scheduling Overhead

A channel send or mutex lock may cause a goroutine to be descheduled. The memory model does not guarantee that the scheduler will flush all writes before parking, but the implementation does. Relying on that is fragile – always use explicit synchronisation.

## 7. Best Practices

1. **Never implement lock‑free code unless you have proof (via `go test -race` and benchmarks) that a mutex is a bottleneck.** Even then, use `sync/atomic` only for simple counters, flags, or sequentially consistent compare‑and‑swap patterns.
2. **Prefer channels for communication, mutexes for shared state.** Channels give stronger happens‑before guarantees (send happens‑before receive) and are more idiomatic.
3. **Run every concurrent test with `-race`.** The race detector understands the memory model and reports violations. It has a small runtime overhead (≈10x) but is invaluable.
4. **Use `sync.Once` for lazy initialisation** – it wraps the double‑checked locking pattern correctly, including the necessary memory barriers.
5. **Document synchronisation expectations.** If your function returns a pointer to an internal object that may be mutated concurrently, write it in the doc comment. Example: `// GetStats returns a snapshot; callers must not modify it.`
6. **Treat `sync/atomic` as a low‑level tool.** Never expose atomic variables in a public API without also exposing the ordering guarantees.
7. **Avoid `runtime.Gosched()`.** It does **not** create a synchronisation edge – it merely yields the processor. Use proper primitives.

## 8. Examples

### Example 1: Correct Lazy Initialisation with `sync.Once`

```go
package cache

import "sync"

var (
    once   sync.Once
    client *http.Client
)

func HTTPClient() *http.Client {
    once.Do(func() {
        // This function runs exactly once.
        // All writes inside happen‑before any call to HTTPClient() returns.
        client = &http.Client{
            Timeout: 10 * time.Second,
            Transport: &http.Transport{
                MaxIdleConns: 100,
            },
        }
    })
    return client
}
```

**Why this works**: `sync.Once.Do` uses atomic loads and a mutex internally, guaranteeing that any write inside `f()` happens‑before any observer sees the `once.Do` call return.

### Example 2: Broken Double‑Checked Locking (Don’t do this)

```go
type Config struct {
    data map[string]string
}

var (
    config     *Config
    configInit int32 // 0 = not started, 1 = done
    configMu   sync.Mutex
)

// BROKEN – do NOT use this pattern.
func GetConfigBroken() *Config {
    if atomic.LoadInt32(&configInit) == 0 { // fast path
        configMu.Lock()
        if atomic.LoadInt32(&configInit) == 0 {
            c := &Config{data: make(map[string]string)}
            // ... populate config ...
            config = c
            atomic.StoreInt32(&configInit, 1)
        }
        configMu.Unlock()
    }
    return config
}
```

This is still subtly broken because the compiler may reorder the stores inside the critical section. Even with atomics, there is no happens‑before edge between the store to `config` and the store to `configInit` across the two different atomic operations. **Use `sync.Once`.**

### Example 3: Using the Race Detector to Find a Data Race

```go
// race.go
package main

var counter int

func main() {
    go func() { counter++ }()
    go func() { counter++ }()
}
```

Run:
```bash
$ go run -race race.go
==================
WARNING: DATA RACE
Write at 0x... by goroutine 7:
  main.main.func2()
      /tmp/race.go:8 +0x...
Previous write at 0x... by goroutine 6:
  main.main.func1()
      /tmp/race.go:7 +0x...
==================
```

The detector points exactly to the two unsynchronised writes.

## 9. Summary & Exercises

### Summary

- Go’s memory model is **SC‑DRF** – write data‑race‑free code and execution appears sequentially consistent.
- Synchronisation primitives (`sync.Mutex`, `sync/atomic`, channels, `sync.Once`, `sync.WaitGroup`) create explicit **happens‑before** edges.
- Data races produce defined but nondeterministic results; use `-race` to detect them.
- Avoid weak memory model tricks – Go provides no relaxed atomics (all atomics are sequentially consistent).
- Prefer channels and mutexes over lock‑free code; only optimise with atomics when profiling proves contention.

### Exercises

**Exercise 1: Happens‑Before Analysis**

Given the following program, identify all happens‑before edges. Is there a data race? If yes, which line(s)? Explain using the Go memory model.

```go
var a, b int

func f() {
    a = 1          // (1)
    b = 2          // (2)
}

func g() {
    for b != 2 {   // (3)
        runtime.Gosched()
    }
    fmt.Println(a) // (4)
}

func main() {
    go f()
    go g()
    time.Sleep(time.Second)
}
```

**Exercise 2: Implement a Thread‑Safe Lazy Initialiser Without `sync.Once`**

Use `sync/atomic` and a mutex to implement a `Lazy[T any]` struct with a `Get(init func() T) T` method. Ensure it is correct according to the memory model. Then compare its performance with `sync.Once` using a benchmark.

**Exercise 3: Find the Race with the Detector**

Write a program that spawns 100 goroutines, each incrementing a shared `int64` 1000 times using `atomic.AddInt64`. Now replace the atomic with a plain `x++`. Run both with `-race`. Then add a mutex to protect the non‑atomic version. Measure the performance of all three approaches (`atomic`, `mutex`, `unsynchronised`) using `go test -bench`. Explain the relative costs.

---

**“Aha!” Moment**: The Go memory model is not something you implement *to* – it is a *contract* that the runtime keeps *for* you, as long as you use the documented synchronisation primitives. Attempting to be “clever” (e.g., double‑checked locking, assuming goroutine exit sync) is where subtle, production‑only bugs are born. The race detector is your best friend.
