## Chapter 26: Synchronization Primitives

Go’s mantra — *“share memory by communicating”* — often leads engineers to believe channels are the only concurrency tool they need. Channels are brilliant for orchestrating communication and ownership transfer, but they are not a universal solution. When multiple goroutines share a mutable state **without** transferring ownership, channels become awkward and inefficient. That’s where the `sync` package steps in, offering low-level primitives that let you protect data directly. Used correctly, they unlock performant, clear, and safe concurrent code; used poorly, they introduce races, deadlocks, and subtle corruption. This chapter dissects `Mutex`, `RWMutex`, `WaitGroup`, `Cond`, and the `atomic` package, and it equips you with a systematic approach for picking between channels and locks, plus the indispensable race detector.

---

### 1. Basic Usage

Let’s start with the syntax that gets your code running.

**sync.Mutex** protects a critical section. The zero value is ready to use.

```go
var mu sync.Mutex
var counter int

func increment() {
    mu.Lock()
    counter++
    mu.Unlock()
}
```

**sync.RWMutex** allows multiple concurrent readers but only one writer. Use `RLock`/`RUnlock` for reads.

```go
var rw sync.RWMutex
var data map[string]string

func get(key string) string {
    rw.RLock()
    v := data[key]
    rw.RUnlock()
    return v
}

func set(key, value string) {
    rw.Lock()
    data[key] = value
    rw.Unlock()
}
```

**sync.WaitGroup** waits for a collection of goroutines to finish.

```go
var wg sync.WaitGroup
for i := 0; i < 5; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        // do work
    }(i)
}
wg.Wait()
```

**sync.Cond** is a condition variable that lets goroutines wait for or announce an event. You create it with a `Locker` (usually a `*Mutex`).

```go
var mu sync.Mutex
cond := sync.NewCond(&mu)
ready := false

// waiter
go func() {
    mu.Lock()
    for !ready {
        cond.Wait()
    }
    // proceed
    mu.Unlock()
}()

// signaler
mu.Lock()
ready = true
cond.Signal()   // or cond.Broadcast()
mu.Unlock()
```

**sync/atomic** provides lock‑free operations on integers and pointers. These are true atomic instructions.

```go
var ops int64
atomic.AddInt64(&ops, 1)
n := atomic.LoadInt64(&ops)
atomic.StoreInt64(&ops, 0)
swapped := atomic.CompareAndSwapInt64(&ops, 10, 20)
```

All primitives are safe for concurrent use by design. They require no initialization beyond their zero values (`sync.Mutex{}`, `sync.WaitGroup{}`), a deliberate simplicity that prevents a class of startup bugs.

---

### 2. Under the Hood

Understanding the runtime machinery behind these primitives helps you anticipate behavior under contention and reason about performance.

**Mutex** is implemented as a state field inside `runtime.mutex`. It uses a hybrid approach:

- **Fast path:** an atomic compare‑and‑swap (CAS) tries to lock an uncontended mutex without entering the kernel.
- **Spinning:** if the mutex is held, the goroutine spins for a short period (active waiting) hoping the holder releases it quickly. This avoids the overhead of a context switch when the critical section is short.
- **Semaphore sleep:** if spinning doesn’t acquire the lock, the goroutine calls `runtime.semacquire`, which parks the goroutine on a semaphore queue. The runtime uses a FIFO semaphore to ensure fairness, preventing starvation (Go 1.18+ refined this further with a new mutex fairness policy).

The zero‑value mutex internally holds the initial state 0 (unlocked). There is no constructor, no “init” function — the zero value **is** the ready state. Copying a mutex after first use is catastrophic, as the internal state descriptor travels with it.

**RWMutex** extends the concept. It embeds a `Mutex` for writer‑side locking and maintains reader count and a reader‑writer semaphore. A writer arriving must wait until all readers leave; incoming readers are blocked until the writer finishes. This design prevents writer starvation to some extent but does not fully guarantee it — bursts of readers can still delay a writer indefinitely in extreme cases.

**WaitGroup** is a thin wrapper around a 64‑bit atomic counter and a semaphore. `Add` increments the counter atomically. `Done` decrements. When the counter hits zero, a single `runtime.semrelease` wakes all waiting goroutines. The internal state (counter and waiters) is packed into a 64‑bit field and a 32‑bit semaphore, avoiding heap allocation for the group itself.

**Cond** builds on a low‑level runtime semaphore and a wait queue. `Wait()` atomically unlocks the associated `Locker` and parks the goroutine; upon wake‑up it re‑acquires the lock before returning. `Signal` wakes one waiter, `Broadcast` wakes all. Wait loops must recheck the condition because a spurious wakeup can occur — Cond follows the Mesa monitor semantics that Go inherited from Plan 9’s synchronization primitives.

**Atomic operations** are backed by hardware instructions (e.g., `LOCK CMPXCHG` on x86, `ldrex/strex` on ARM) when the value is properly aligned. The `sync/atomic` package bypasses the scheduler; there is no blocking, no spinning, and no interaction with the GC. Because of this, they’re incredibly fast but offer no memory ordering guarantees beyond the sequentially‑consistent atomics the package provides by default (Go 1.19 introduced `atomic.Int64` and friends with clearer semantics). The compiler treats these as compiler barriers, preventing reordering around them.

---

### 3. Why This Design?

Why does Go, a language that preaches “communicate by sharing memory,” offer such classic shared‑memory primitives? Because not every problem is a communication problem. When a cache, a configuration map, or a connection pool is genuinely shared across many goroutines with no single owner, wrapping it in a channel forces you to invent artificial goroutine owners and message protocols. That leads to complex, slower code.

Go’s design choices reflect a philosophy of **giving you the right tool for the right job, with minimal ceremony**:

- **Mutex as a struct, not a handle:** you can embed a `sync.Mutex` directly in your struct to protect its fields. There’s no separate constructor or error handling because a zero mutex is ready. Contrast with C++ `std::mutex` (must not be moved) or Java’s `ReentrantLock` (requires explicit instantiation). The Go way reduces boilerplate and eliminates the question “did I forget to init this lock?”

- **No reentrancy:** Go mutexes are not reentrant. The same goroutine calling `Lock` twice deadlocks. This is intentional. Reentrant locks hide concurrency bugs; a function that re‑locks a mutex it already holds often indicates a poorly factored lock scope. By forbidding reentry, Go forces you to keep locking scopes clear and narrow.

- **WaitGroup as a simple counter:** instead of higher‑level abstractions like `CountDownLatch` or futures, Go gives you a bare‑bones counter with semaphore‑based waiting. You manage the count yourself. This pushes responsibility to the programmer but avoids framework complexity and hidden thread‑pool dependencies.

- **Cond over higher‑level event primitives:** Go’s standard library offers `sync.Cond` rather than an asynchronous event or promise. It’s a direct, low‑level building block. If you need a bounded blocking queue, you can build it with `Cond` or channels. The team deliberately left `Cond` underpowered (no integrated waiting with timeout, no spurious‑wake‑up protection) to keep the runtime simple and encourage the use of channels for most signalling. `Cond` exists for the rare cases where you need explicit condition variables and cannot use channels.

- **Atomic package separation:** atomics live in `sync/atomic`, not in the language itself. This draws a sharp line: you must consciously opt into lock‑free programming. Hiding atomics behind language constructs (like Java’s `volatile` or C’s `_Atomic`) would blur the cost model. In Go, you import the package and use them with full awareness.

The guiding rule is **simplicity over framework**. Each primitive is minimal, composable, and leaves policy decisions (like fair scheduling or timeout handling) to you or higher‑level libraries.

---

### 4. Competing Approaches

When you come from another ecosystem, mapping your mental model to Go’s toolbox avoids misuse.

**Java / C#**
- Java offers intrinsic locks (`synchronized`), `ReentrantLock`, `ReadWriteLock`, `CountDownLatch`, `CyclicBarrier`, `Phaser`, and a rich `java.util.concurrent.atomic` package. Go’s approach is deliberately spartan. There is no equivalent to `synchronized` on a method; you embed a `Mutex` and explicitly lock. That explicitness prevents accidental lock‑retention across method boundaries.
- Java’s `synchronized` is reentrant; Go’s mutex is not. If you attempt to replicate a Java pattern where a method calls another synchronized method on the same object, you’ll deadlock in Go. Restructure your code so that the lock is held for the entire composed operation, or expose internal lock‑free helpers.
- `CountDownLatch` maps directly to `sync.WaitGroup`, but Go’s `WaitGroup` is simpler (no count‑down‑reset capability). If you need to reuse the barrier, allocate a new `WaitGroup`.
- Java’s `Condition` (from `Lock`) mirrors Go’s `Cond`, including the need for a while‑loop to guard against spurious wakeups.

**Python**
- Python’s `threading.Lock` is reentrant in its `RLock` variant, but `Lock` is non‑reentrant like Go. The GIL makes many Python locks unnecessary for data races (though they still protect against concurrent bytecode interleaving). Go’s goroutines are preemptive and truly parallel, so locks are mandatory for shared mutable state.
- Python condition variables (`threading.Condition`) work identically in spirit. Go’s `Cond` has no `wait_for()` predicate; you must write the loop yourself.
- Python’s `concurrent.futures` and asyncio replace channels; Go builds concurrency into the language, making WaitGroup and channel composition more natural.

**C++**
- C++ `std::mutex` and `std::shared_mutex` (C++17) are the closest analogues. However, C++ mutexes cannot be copied or moved; Go’s mutexes can be embedded in structs and the structs can be passed by pointer. That’s a subtle advantage in composition.
- C++ provides `std::lock_guard` and `std::unique_lock` for RAII. Go lacks destructors, so the idiom is `defer mu.Unlock()` — just as safe if you remember it. The absence of RAII is compensated by a culture of `defer` immediately after `Lock`.
- C++ atomics are a deep language feature with multiple memory orders. Go’s atomics are sequentially consistent by default (except for a few weaker operations added in 1.19). This prevents accidental reordering bugs but can be slower on weak‑ordered architectures. You pay for safety unless you explicitly use the slower `atomic.StoreInt64Release` (expert territory).

**Rust**
- Rust’s `std::sync::Mutex` wraps the protected data; you cannot access the data without locking. Go’s mutex is an external lock; the data is just a separate field. That means Go’s compiler cannot guarantee you’ve locked the mutex before accessing the field — the race detector is your safety net.
- Rust’s `RwLock` and `Condvar` map closely, but Rust’s ownership prevents sending non‑`Send` types across threads, eliminating many race conditions at compile time. Go places this burden on you and the race detector.
- Rust atomics have explicit `Ordering`, while Go’s have simpler defaults. In both cases, misuse leads to subtle bugs, but Go’s lighter touch makes atomics more approachable for simple counters.

The theme across languages: Go picks the **minimal viable primitive**, removes “conveniences” that mask bugs (like reentrancy), and trusts you to enforce correctness with explicit locking patterns and tools like the race detector.

---

### 5. Common Mistakes

Even seasoned engineers trip over these.

**Copying a mutex.** A `sync.Mutex` must never be copied after first use. If your struct contains a mutex, always pass it by pointer or store it behind a pointer. Copying duplicates the internal state (waiters, semaphore), leading to deadlocks or unsynchronized access. `go vet` catches many copied‑mutex scenarios, but be vigilant when embedding mutexes in value‑type structs.

**Forgetting to unlock, especially in error paths.** In C++ you have RAII; in Go you have `defer`. Put `defer mu.Unlock()` immediately after `mu.Lock()` unless you need to unlock early. But `defer` runs at function exit; if you hold a lock across a long function that doesn’t exit quickly, you’ll throttle concurrency. Ensure locked sections are small; if your function does locked work, then a lot of unlocked work, consider splitting it so you can unlock before the heavy part.

**Holding a lock during I/O or blocking channel operations.** If you hold a mutex while reading from a network socket or sending on a channel, you can deadlock with other goroutines waiting for the mutex. Lock only around the minimal state mutation.

**RWMutex misuse.** Using `RWMutex` when writes are frequent causes writer starvation and kills throughput. Profile before choosing it. Also, don’t call `RLock` inside a critical section already holding a write lock — it deadlocks.

**WaitGroup pitfalls.**
- Calling `wg.Add(1)` inside the goroutine instead of before it starts. The goroutine may start running before `Add` is called, and `Wait` might return prematurely. Always `Add` before launching the goroutine.
- Negative counter: `Done` more times than `Add` causes a panic.
- Reusing a `WaitGroup` before `Wait` returns: if you call `Wait`, then later call `Add` and `Wait` again, there’s a race. The safe pattern is to never reuse a `WaitGroup` once `Wait` has begun.

**Cond gotchas.**
- Not wrapping `Wait` in a `for` loop. Spurious wakeups happen, and signals can be lost. Always check the condition after waking.
- Calling `Wait` without holding the lock. This panics.
- Forgetting `Signal` or `Broadcast` after changing the condition. The waiting goroutine may sleep forever.
- Using `Cond` when a buffered channel of size 1 works perfectly. `Cond` is powerful but error‑prone; reserve it for when you truly need multi‑waiter broadcasting.

**Atomic vs. mutex for compound operations.** `atomic.AddInt64` is atomic for the addition, but if you need to load, conditionally modify, and store, an atomic CAS loop is required. Mistakenly using separate `Load` and `Store` introduces a data race. If the logic gets complex, a mutex is clearer and often fast enough.

**Ignoring the race detector.** Writing concurrent code without regularly running `go test -race` is like C programming without Valgrind. The detector finds races at runtime, but only on exercised code paths. Never trust concurrent code that hasn’t passed the race detector.

---

### 6. Performance Considerations

Locking isn’t free, but the cost varies dramatically with contention.

**Mutex overhead.** In the uncontended case, lock/unlock with a `sync.Mutex` costs ~15–20 ns on modern hardware (essentially one atomic CAS plus a compiler fence). Under light contention, spinning absorbs the cost without OS involvement. Under high contention, goroutines park and unpark, which costs multiple microseconds and incurs scheduler overhead. The fair mutex (Go 1.18+) trades a small uncontended cost for better tail latency under heavy load.

**RWMutex overhead.** An uncontended `RLock` is slightly cheaper than a full `Lock` because it only atomically increments a reader count. However, a writer must wait for all readers to drain, which can cause latency spikes. If your read‑lock sections are not extremely short, a writer can be delayed long enough to hurt throughput.

**Atomic operations.** A single atomic add/load is ~5–10 ns, often cheaper than a mutex acquisition because it avoids any potential sleeping. However, atomics do not protect a block of statements; they protect a single variable. If your logic spans multiple instructions, a mutex might be faster than a CAS retry loop under contention because the mutex parks the goroutine instead of burning CPU.

**Lock granularity.** Coarse‑grained locks cause contention under parallel workloads; fine‑grained locks increase complexity and memory footprint. Profile‑guided optimization is essential. A common pattern is to split a single global mutex into a per‑shard mutex (as in `sync.Map`’s read‑mostly paths).

**False sharing.** If two goroutines update different variables that reside on the same cache line, the CPU’s cache‑coherency protocol bounces the line between cores, even without a data race. Padding fields to align on cache line boundaries (64 bytes) with `[64]byte` can help in performance‑critical code.

**Channels vs. mutexes performance.** For simple state protection, a mutex is generally faster and allocates less than a channel. A buffered channel involves a mutex internally (the channel’s own lock) plus memory for the buffer and a goroutine if you use it as a one‑slot owner. Use channels when the communication *is* the coordination (e.g., passing data to a dedicated worker), not when you just need to guard a shared map.

**Benchmarking lock overhead.** Use `go test -bench` with `-cpu` profiles and the `sync`‑specific contention profiling (`runtime.SetMutexProfileFraction`) to see where your goroutines are stuck. The `pprof` mutex profile reveals hot locks.

---

### 7. Best Practices

The “idiomatic Go” way to use synchronization primitives revolves around clarity, guard clauses, and recognizing the channel/mutex boundary.

- **Guard locks with `defer` — immediately.** After `mu.Lock()`, the very next line should be `defer mu.Unlock()` if the rest of the function is the critical section. If you unlock early inside the function, do so explicitly but ensure no early return skips it. `defer` protects all future code modifications from accidentally missing an unlock.

- **Mutex ownership belongs to the struct.** Embed the `sync.Mutex` in the struct whose fields it protects. This makes the lock’s scope obvious and prevents it from being misused elsewhere.

  ```go
  type Cache struct {
      mu    sync.RWMutex
      items map[string]Item
  }
  ```

- **Keep critical sections short.** If a function holds a lock for five milliseconds while computing a hash, you’re serializing the entire concurrent workload. Do the compute outside the lock, then lock only to swap the result.

- **Prefer RWMutex when reads vastly outnumber writes and read sections are quick.** But always confirm with benchmarks, not assumptions.

- **Use atomic for simple counters and flags.** For a simple incrementing request counter, `atomic.AddInt64` is cleaner and faster than a mutex.

- **Channels for coordination, mutexes for state.** If you’re transferring ownership of data (e.g., a worker pool sending results), use a channel. If multiple goroutines read/write a shared map and no single goroutine owns it, use a mutex. An anti‑pattern is a channel of length 1 used as a lock: it’s slower, allocates, and obscures intent.

- **Avoid `sync.Cond` unless you have no alternative.** Before reaching for `Cond`, see if a channel of struct{} or a simple `sync.Mutex` with a boolean flag suffices. Often, a goroutine that polls a condition periodically wrapped in a `select` with a timer is clearer and less bug‑prone.

- **The race detector is a mandatory part of your test suite.** Add `//go:build race` for tests that intentionally stress concurrent paths, and run your CI with `go test -race -count=1 ./...`. Fix every race it finds, even if it seems benign; “benign” data races are undefined behavior in Go’s memory model.

- **When you embed a mutex, don’t expose it.** Keep the mutex unexported. Export methods that safely access the data, ensuring the locking logic is contained.

---

### 8. Examples

#### Thread‑safe Cache with RWMutex

A simple in‑memory cache demonstrating reader/writer locks.

```go
type Cache struct {
    mu    sync.RWMutex
    store map[string]any
}

func NewCache() *Cache {
    return &Cache{store: make(map[string]any)}
}

func (c *Cache) Get(key string) (any, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    val, ok := c.store[key]
    return val, ok
}

func (c *Cache) Set(key string, val any) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.store[key] = val
}
```

#### Worker Pool with WaitGroup

Launch a fixed number of workers, each processing jobs until the input channel closes.

```go
func processJobs(jobs <-chan Job, workerCount int) {
    var wg sync.WaitGroup
    for i := 0; i < workerCount; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                job.Execute()
            }
        }()
    }
    wg.Wait()
}
```

#### Bounded Buffer Using Cond (Producer‑Consumer)

A classic condition variable scenario: a fixed‑size buffer where producers wait if full, consumers wait if empty.

```go
type BoundedBuffer struct {
    mu    sync.Mutex
    cond  *sync.Cond
    buf   []int
    cap   int
}

func NewBoundedBuffer(capacity int) *BoundedBuffer {
    b := &BoundedBuffer{
        cap: capacity,
        buf: make([]int, 0, capacity),
    }
    b.cond = sync.NewCond(&b.mu)
    return b
}

func (b *BoundedBuffer) Put(item int) {
    b.mu.Lock()
    defer b.mu.Unlock()
    for len(b.buf) == b.cap {
        b.cond.Wait()
    }
    b.buf = append(b.buf, item)
    b.cond.Signal()
}

func (b *BoundedBuffer) Get() int {
    b.mu.Lock()
    defer b.mu.Unlock()
    for len(b.buf) == 0 {
        b.cond.Wait()
    }
    item := b.buf[0]
    b.buf = b.buf[1:]
    b.cond.Signal()
    return item
}
```

#### Lock‑free Counter with Atomic

A simple request counter that doesn’t need a lock.

```go
type Counter struct {
    n atomic.Int64
}

func (c *Counter) Inc() {
    c.n.Add(1)
}

func (c *Counter) Value() int64 {
    return c.n.Load()
}
```

#### Detecting a Race with the Race Detector

Take this buggy code:

```go
func main() {
    var wg sync.WaitGroup
    count := 0
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            count++ // race!
            wg.Done()
        }()
    }
    wg.Wait()
    fmt.Println(count)
}
```

Running `go run -race main.go` produces:

```
WARNING: DATA RACE
Read at 0x00c0000b4010 by goroutine 7:
  main.main.func1()
      /tmp/main.go:12 +0x4c

Previous write at 0x00c0000b4010 by goroutine 6:
  main.main.func1()
      /tmp/main.go:12 +0x62
```

The fix: either protect `count` with a mutex, or use an `atomic.Int64`. The race detector pinpoints the exact lines and access types, making debugging fast.

---

### 9. Summary & Exercises

Go’s synchronization primitives complement channels, giving you complete control over shared memory. `Mutex` and `RWMutex` protect critical sections, `WaitGroup` collects task completions, `Cond` handles event broadcasting, and `atomic` delivers lock‑free safety for single variables. The golden rule: **if you’re coordinating communication, use channels; if you’re protecting state, use locks**. Always run the race detector, keep locked sections short, and embed mutexes with their protected data.

**Exercises**

1. **Thread‑safe LRU Cache:** Implement an LRU cache with a mutex. The cache should support `Get(key string) (value any, ok bool)` and `Put(key string, value any)` with a maximum capacity. When capacity is exceeded, evict the least recently used item. Ensure all operations are safe for concurrent use. Write a test that fires hundreds of concurrent goroutines at it and passes the race detector.

2. **Counting Semaphore from Primitives:** Implement a counting semaphore (allow exactly N concurrent accesses) using two different approaches: first using a buffered channel of size N, then using `sync.Mutex` and `sync.Cond` (or a `WaitGroup` / atomic counter). Compare the readability, error‑proneness, and performance of each under high contention. Which is more idiomatic Go? Why?

3. **Race Detective Work:** The following program has a subtle data race that the race detector might not catch on a single run because the race depends on timing. Study the code, hypothesize the race, then write a test that uses the race detector and a `sync.WaitGroup` to reliably trigger and capture the race. Fix it.

   ```go
   type Config struct {
       values map[string]string
   }

   func (c *Config) Update(k, v string) {
       c.values[k] = v
   }

   func (c *Config) Get(k string) string {
       return c.values[k]
   }
   // Multiple goroutines call Update and Get on the same *Config.
   ```
