# Chapter 26: Synchronization Primitives

If goroutines are the actors in a concurrent Go program, then **synchronization primitives** are the traffic lights, barriers, and handshake protocols that prevent chaos. While channels are Go’s headline concurrency feature (“Share memory by communicating”), the standard library provides a full set of low‑level primitives—`sync.Mutex`, `sync.RWMutex`, `sync.WaitGroup`, `sync.Cond`, and `sync/atomic`—for cases where channels add unnecessary overhead or obscure the logic.

This chapter assumes you understand race conditions, goroutine scheduling, and the basics of the Go memory model. We’ll focus on *when* to reach for each primitive, how they behave under the hood, and why Go’s design choices around locking are deliberately modest.

---

## Basic Usage

All primitives live in the `sync` package (except atomics, which are in `sync/atomic`). They are designed to be used without initialisation (zero values are usable for `Mutex`, `RWMutex`, `WaitGroup`, `Cond`, and `atomic.Value`).

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	"time"
)

// 1. Mutex: protect a shared counter
type Counter struct {
	mu    sync.Mutex
	value int
}

func (c *Counter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.value++
}

func (c *Counter) Val() int {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.value
}

// 2. RWMutex: many readers, occasional writers
type SafeMap struct {
	mu sync.RWMutex
	m  map[string]string
}

func NewSafeMap() *SafeMap {
	return &SafeMap{m: make(map[string]string)}
}

func (sm *SafeMap) Get(key string) (string, bool) {
	sm.mu.RLock()
	defer sm.mu.RUnlock()
	v, ok := sm.m[key]
	return v, ok
}

func (sm *SafeMap) Set(key, val string) {
	sm.mu.Lock()
	defer sm.mu.Unlock()
	sm.m[key] = val
}

// 3. WaitGroup: wait for N goroutines to finish
func worker(id int, wg *sync.WaitGroup) {
	defer wg.Done() // decrement counter
	fmt.Printf("worker %d starting\n", id)
	time.Sleep(time.Millisecond * 100)
	fmt.Printf("worker %d done\n", id)
}

// 4. Atomic operations (lock‑free)
var hits atomic.Int64

func handleRequest() {
	hits.Add(1) // no lock required
}

func main() {
	// Mutex example
	c := Counter{}
	var wg sync.WaitGroup
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			c.Inc()
		}()
	}
	wg.Wait()
	fmt.Println("counter:", c.Val()) // 1000

	// WaitGroup example
	var wg2 sync.WaitGroup
	for i := 1; i <= 5; i++ {
		wg2.Add(1)
		go worker(i, &wg2)
	}
	wg2.Wait() // blocks until all five call Done
	fmt.Println("all workers finished")

	// Atomic example
	for i := 0; i < 100; i++ {
		go handleRequest()
	}
	time.Sleep(time.Millisecond)
	fmt.Println("hits:", hits.Load()) // 100 (with a small sleep to let goroutines run)
}
```

> **Note:** `sync.Cond` (condition variable) is intentionally omitted from this “Basic Usage” section because it’s rarely needed and often misused. We’ll mention it later for completeness.

---

## Under the Hood

### Mutex Implementation

A `sync.Mutex` in Go is **not** a simple spinlock. The runtime implements a two‑stage queue:

1. **Normal mode** – New goroutines compete for the lock via a spin‑wait (a few dozen iterations) before parking. This reduces context switches when the lock is held only briefly.
2. **Starvation mode** – If a goroutine fails to acquire the lock for longer than 1ms, the mutex enters starvation mode. In this mode, the lock is handed directly to the next waiting goroutine (FIFO), preventing unbounded latencies.

Internally, the mutex uses an `int32` field (`state`) with bits tracking:
- Locked flag (whether the mutex is held)
- Woken flag (a parked goroutine has been woken)
- Starving flag
- Waiter count (the number of goroutines waiting in the queue)

Lock acquisition uses atomic compare‑and‑swap (CAS) operations. The runtime’s scheduler integration means that a goroutine that cannot acquire the lock will be parked (not spinning the CPU) and later unparked when the lock is released.

### RWMutex

`sync.RWMutex` builds on `Mutex` but adds a reader counter. It allows:
- Multiple simultaneous readers (RLock / RUnlock)
- Exclusive writers (Lock / Unlock)

The implementation tracks the number of active readers. When a writer attempts to lock, it blocks *future* readers and waits for existing readers to drain.

### WaitGroup Internals

A `WaitGroup` holds a `state` field (64‑bit) combining:
- Counter (number of remaining goroutines)
- Waiter count (number of `Wait` calls blocked)

The `Add` and `Done` methods manipulate the counter atomically. When the counter reaches zero while waiters exist, the runtime wakes all waiting goroutines (implemented as a semaphore).

### Atomic Operations (`sync/atomic`)

The `atomic` package uses CPU‑specific instructions:
- `LOCK CMPXCHG` on x86
- `LDREX/STREX` on ARM

These instructions guarantee that the operation is *indivisible* across all cores. Go exposes typed atomic values (`atomic.Int64`, `atomic.Bool`, etc.) that eliminate common errors like forgetting `LOAD`/`STORE` consistency.

### Race Detector Integration

All `sync` primitives are instrumented when building with `-race`. The race detector inserts extra memory access events and tracks happens‑before relationships. A correct use of `Lock`/`Unlock` tells the race detector that accesses inside the critical section do not race.

---

## Why This Design?

Go’s concurrency philosophy is “Share memory by communicating,” so why provide locks at all?

1. **Performance** – A channel send/receive involves runtime scheduling, memory allocation (for buffered channels), and potential goroutine handoff. A mutex can be as cheap as ~10ns under low contention, while an unbuffered channel send is ~50ns and involves more work. For high‑frequency, fine‑grained protection (e.g., updating a map from many goroutines), a mutex is measurably faster.

2. **Simplicity in specific domains** – Protecting a `map` or a simple counter with a mutex is often easier to read than modelling that state as a channel‑driven goroutine (a “server” goroutine that owns the state). The latter is elegant but requires extra ceremony.

3. **Existing patterns** – Many programmers come from lock‑based concurrency. Go provides these primitives as a *responsible escape hatch*, not as the primary recommendation.

4. **Atomicity for trivial operations** – `sync/atomic` allows lock‑free algorithms for statistics, flags, and reference counting. Channels cannot provide the same sub‑goroutine granularity.

**The “Aha!” moment**: Locks in Go are *explicit* and *unforgiving*. There’s no `synchronized` method or implicit monitor – you always see where the lock is acquired and released. This aligns with Go’s value of clarity over magic.

---

## Competing Approaches

| Language | Primary Primitive | How Go Differs |
|----------|------------------|----------------|
| **Java** | `synchronized` (intrinsic locks), `ReentrantLock`, `ReadWriteLock`, `volatile` | Go has no intrinsic lock for every object. Locks are first‑class values (`sync.Mutex`) with no reentrancy. Java’s `volatile` is replaced by `atomic` loads/stores and `sync/atomic` primitives. |
| **C++** | `std::mutex`, `std::shared_mutex`, `std::atomic` | C++ gives you the same building blocks but with *undefined behavior* for data races. Go defines the memory model and has a race detector to catch violations. C++ supports reentrant mutexes (`std::recursive_mutex`); Go deliberately does not. |
| **Rust** | `Mutex<T>`, `RwLock<T>` (wrapped around data), `Atomic*` types | Rust’s type system *encodes* locking: a `Mutex<i32>` can only be accessed by calling `lock()`, which returns a guard that releases when dropped. Go has no such compile‑time guarantee – you can forget to lock. However, Go’s race detector catches many errors at test time. |
| **Python** | `threading.Lock`, `asyncio.Lock` | Python’s GIL means that a `Lock` only prevents *thread* switches, not true parallelism. Go’s locks are for parallel execution on multiple cores. Python’s lock performance is much worse (heavy context). |

**Go’s unique stance**: Provide both channels *and* locks, but strongly steer towards channels for ownership and orchestration. The standard library uses locks internally (e.g., `sync.Map`, `http.Server`) but exports channels for user‑facing APIs.

---

## Common Mistakes

### 1. Copying a `sync.Mutex`

```go
type Counter struct {
    mu sync.Mutex
    v  int
}

func (c Counter) Inc() { // ❌ value receiver – copies the mutex!
    c.mu.Lock()
    defer c.mu.Unlock()
    c.v++
}
```

The mutex is copied, so the lock operates on a different instance. **Always use pointer receivers when the struct contains a `sync.Mutex` (or any `sync` type)**.

### 2. Forgetting to Unlock (especially in error paths)

```go
func update(data []byte) error {
    mu.Lock()
    if len(data) == 0 {
        return errors.New("empty") // ❌ returns without unlock
    }
    // ... modify shared state
    mu.Unlock()
    return nil
}
```

**Fix:** Use `defer mu.Unlock()` immediately after locking, even in short functions.

### 3. Deadlock from Lock Ordering

```go
var muA, muB sync.Mutex

// goroutine 1
muA.Lock()
muB.Lock() // waits for B
// ...
muB.Unlock()
muA.Unlock()

// goroutine 2
muB.Lock()
muA.Lock() // waits for A, deadlock
```

Go’s runtime does **not** detect deadlocks at runtime unless all goroutines are blocked (then it panics). Fix by using a consistent lock ordering or replacing with a channel handoff.

### 4. RWMutex Write Starvation

Under high read load, a writer may never acquire the lock because new readers keep arriving. Go’s `RWMutex` implementation gives writers priority in newer versions (Go 1.18+), but you can still starve if you hold `RLock` for a long time. **Solution:** Keep read locks short, or use a `Mutex` if writes are frequent.

### 5. WaitGroup Misuse – `Add` Inside the Goroutine

```go
var wg sync.WaitGroup
for i := 0; i < 10; i++ {
    go func() {
        wg.Add(1) // ❌ too late – Wait may already be called
        defer wg.Done()
        // work
    }()
}
wg.Wait()
```

Always call `wg.Add()` before spawning the goroutine, or use the pattern `wg.Add(1); go func() { defer wg.Done(); ... }()`.

### 6. Mixing Atomic and Non‑atomic Access

```go
var counter int64 // accessed atomically
// In one goroutine
atomic.AddInt64(&counter, 1)
// In another goroutine
val := counter // ❌ non‑atomic read – race!
```

If you touch a variable with `atomic` operations, **all** accesses must be atomic. Use `atomic.LoadInt64` and `atomic.StoreInt64` consistently.

### 7. `sync.Cond` Misuse – Signals Without Lock

`sync.Cond` requires that you hold the associated `Locker` (usually a `Mutex`) before calling `Wait`. Forgetting this leads to lost wakeups or race conditions. Modern Go code almost always prefers channels for signalling.

---

## Performance Considerations

| Primitive | Approximate cost (low contention) | Contention scaling |
|-----------|-----------------------------------|---------------------|
| `atomic.Add` | 2–5 ns | Perfect – lock‑free |
| `sync.Mutex` (uncontended) | 10–15 ns | Co‑operative (spinning then parking) |
| `sync.RWMutex` (read lock) | 12–18 ns | Readers scale linearly until a writer arrives |
| Unbuffered channel send | 40–50 ns | Involves goroutine handoff |
| `sync.WaitGroup` (`Wait` + `Done`) | 20 ns + scheduler cost | Blocking waiters are parked |

**Key metrics:**

- **Cache line bouncing** – When multiple cores constantly lock the same mutex, the cache line containing the mutex state is invalidated on every lock/unlock. This degrades performance drastically (could be 100x slower). Solution: reduce lock frequency or shard the data (e.g., `sync.Map` or a slice of mutexes).

- **False sharing** – Two unrelated variables on the same cache line cause contention. Align variables (e.g., using padding) or rely on Go’s memory layout (the compiler may add padding automatically for `atomic.Int64`).

- **Mutex vs. channel throughput** – For a simple counter, a mutex is ~4x faster than a channel‑based “server” goroutine. However, as critical section size grows (e.g., serializing a 1KB struct), the difference shrinks and maintainability may favour channels.

- **Atomic vs. mutex** – Use atomics for *single* word operations (counters, flags, pointers). For anything more complex (e.g., incrementing a counter and updating a string), a mutex is safer and often fast enough.

**Garbage collection impact:** Locks do not directly affect GC. However, if a lock protects a large heap structure, the GC still scans that memory. Lock contention can indirectly increase GC pressure by forcing more goroutines into runnable state, but the effect is usually small.

---

## Best Practices

1. **Prefer channels for ownership transfer, mutexes for state protection.**  
   Use a mutex when you have a single data structure (like a `map`) that multiple goroutines read/write. Use a channel when a goroutine *owns* the data and others send requests.

2. **Keep lock scope as small as possible.**  
   ```go
   mu.Lock()
   val := m["key"]
   mu.Unlock()
   process(val) // do expensive work outside the lock
   ```

3. **Never copy `sync.Mutex`, `sync.RWMutex`, `sync.WaitGroup`, or `sync.Cond`.**  
   Pass them by pointer. Linter tools (e.g., `go vet`) warn about copying lock values.

4. **Always use `defer` to unlock, even in trivial functions.**  
   This prevents future additions from forgetting to unlock on early returns.

5. **Use `sync/atomic` with typed values (`atomic.Int64`, `atomic.Bool`, `atomic.Pointer[T]`).**  
   They reduce mistakes and are self‑documenting.

6. **Run tests with `-race` in your CI pipeline.**  
   The race detector catches almost all incorrect lock usage, but it does *not* detect deadlocks. For deadlocks, use the blocking profiler (`net/http/pprof`).

7. **Prefer `RWMutex` only when reads dominate writes and read‑side operations are long.**  
   For trivial getters, a simple `Mutex` is often clearer and not much slower.

8. **Don’t use `sync.Cond`. Replace it with channels.**  
   `sync.Cond` is error‑prone; a buffered channel or `select` with `time.Ticker` is nearly always simpler.

9. **Use `WaitGroup` for batch wait, not as a counting semaphore.**  
   For limiting concurrency, use a worker pool with channels or `semaphore.Weighted` (from `golang.org/x/sync`).

10. **Document which fields a mutex protects.**  
    ```go
    type Cache struct {
        mu sync.Mutex
        // mu protects entries and lastCleanup
        entries      map[string]Item
        lastCleanup  time.Time
    }
    ```

---

## Examples

### Example 1: Thread‑Safe Counter with Mutex

```go
type AtomicCounter struct {
    mu    sync.Mutex
    value int
}

func (c *AtomicCounter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}

func (c *AtomicCounter) Load() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.value
}
```

### Example 2: RWMutex for a Read‑Heavy Cache

```go
type Cache struct {
    mu   sync.RWMutex
    data map[string][]byte
}

func (c *Cache) Get(key string) ([]byte, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    val, ok := c.data[key]
    return val, ok
}

func (c *Cache) Set(key string, val []byte) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.data[key] = val
}

func (c *Cache) DeleteExpired(expireFunc func([]byte) bool) int {
    c.mu.Lock()
    defer c.mu.Unlock()
    deleted := 0
    for k, v := range c.data {
        if expireFunc(v) {
            delete(c.data, k)
            deleted++
        }
    }
    return deleted
}
```

### Example 3: WaitGroup with Error Collection

```go
func processAll(items []string) ([]Result, error) {
    var wg sync.WaitGroup
    results := make([]Result, len(items))
    errs := make([]error, len(items))

    for i, item := range items {
        wg.Add(1)
        go func(idx int, it string) {
            defer wg.Done()
            res, err := process(it)
            results[idx] = res
            errs[idx] = err
        }(i, item)
    }
    wg.Wait()

    // Combine errors
    var finalErr error
    for _, err := range errs {
        if err != nil {
            finalErr = errors.Join(finalErr, err)
        }
    }
    return results, finalErr
}
```

### Example 4: Atomic Flag for Cancellation

```go
var stopped atomic.Bool

func worker(ctx context.Context) {
    for {
        if stopped.Load() {
            return
        }
        select {
        case <-ctx.Done():
            return
        default:
            // do work
        }
    }
}

func stop() {
    stopped.Store(true)
}
```

### Example 5: Correct Use of `sync.Once`

`sync.Once` ensures a function runs exactly once, even under high concurrency.

```go
var (
    once     sync.Once
    database *sql.DB
)

func getDB() *sql.DB {
    once.Do(func() {
        db, err := sql.Open("pgx", "postgres://...")
        if err != nil {
            panic(err) // panic once, during initialisation
        }
        database = db
    })
    return database
}
```

---

## Summary & Exercises

**Summary**

- Go provides `sync.Mutex`, `sync.RWMutex`, `sync.WaitGroup`, `sync.Once`, `sync.Cond`, and `sync/atomic` as low‑level concurrency primitives.
- Use mutexes to protect shared state; use channels to communicate ownership.
- Never copy a `sync` value – always pass pointers.
- The race detector (`-race`) is essential for finding data races but does not detect deadlocks.
- Atomic operations are lock‑free and extremely fast, but only for simple operations on a single word.
- `RWMutex` helps read‑heavy workloads but watch for writer starvation.
- `WaitGroup` is the idiomatic way to wait for goroutine completion.

**Key insights to remember**
- Locks are *not* evil in Go – they are often the simplest solution for protecting a map or a counter.
- The “share memory by communicating” mantra is a north star, not a law. Practical Go uses both.
- Explicit locking (no implicit reentrancy) forces clarity: you always know where the lock is held.

---

### Exercises

**Exercise 1: Thread‑safe cache with TTL**  
Implement a cache that supports `Get`, `Set`, and periodic cleanup of expired entries. Use `sync.RWMutex` to allow concurrent reads. Expired entries should be removed during a `Cleanup` goroutine that runs every minute. Ensure that the cleanup does not block reads for longer than necessary.

*Hint:* Use a heap or a priority queue to store expiration times, or simply scan the map (acceptable for modest sizes). Protect the map with the RWMutex, and use `RLock` for `Get`, `Lock` for `Set` and `Cleanup`.

**Exercise 2: Parallel merge sort with WaitGroup**  
Write a merge sort function that splits the slice into two halves, sorts each half in a new goroutine, then merges the results. Use `sync.WaitGroup` to wait for both halves to complete. Compare performance against a sequential sort for slice sizes of 1M integers. What’s the optimal depth before falling back to sequential sort?

*Hint:* Use a threshold (e.g., 5000 elements) below which you sort sequentially to avoid goroutine overhead.

**Exercise 3: Detect and fix a race condition**  
The following code has a data race. Run it with `-race`, identify the problem, and fix it using *both* a mutex and an atomic operation (two separate solutions). Explain which solution you prefer and why.

```go
type Stats struct {
    requests int64
    errors   int64
}

var stats Stats

func recordRequest() {
    stats.requests++
}

func recordError() {
    stats.errors++
}

func printStats() {
    fmt.Printf("requests: %d, errors: %d\n", stats.requests, stats.errors)
}

func main() {
    for i := 0; i < 1000; i++ {
        go recordRequest()
        if i%10 == 0 {
            go recordError()
        }
    }
    time.Sleep(time.Second)
    printStats()
}
```

**Exercise 4: Build a `sync.Map` alternative**  
The standard library’s `sync.Map` is specialised for certain patterns (write‑once, read‑many). Implement your own thread‑safe map that uses `sync.RWMutex` and a plain Go map. Benchmark it against `sync.Map` for:
- Many concurrent reads with occasional writes.
- Many concurrent writes with occasional reads.
- A mix of 50% reads, 50% writes.

*Deliverable:* Write a benchmark using `testing.B` and report the results. Explain under which conditions your implementation is faster, and when `sync.Map` wins.

---

**Answers to exercises are available in the book’s companion repository.** The next chapter, *File & OS Operations*, will move from pure concurrency to interacting with the outside world – files, directories, environment variables, and system processes.
