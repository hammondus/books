## Chapter 33: Runtime & Memory Model

### 1. Basic Usage

The Go memory model defines the conditions under which a goroutine is guaranteed to observe values produced by another goroutine. You don’t call a special API to “enable” it—the model is embedded in how you use synchronization primitives. The most common patterns boil down to three families.

**Happens-before via channels**

A send on a channel happens-before the corresponding receive completes. This guarantees that any writes done before the send are visible to the receiving goroutine after the receive.

```go
var data string
done := make(chan struct{})

go func() {
    data = "hello, gopher"   // (1) write
    close(done)              // (2) send (close is a send)
}()

<-done                       // (3) receive
fmt.Println(data)            // (4) guaranteed to print "hello, gopher"
```

The write at (1) is sequenced before the close at (2). The close (a send) happens-before the receive at (3). The receive happens-before the print at (4). Therefore, (1) happens-before (4) and the write is visible.

**Happens-before via sync.Mutex**

An unlock on a `sync.Mutex` happens-before a subsequent lock that succeeds. The critical section idiom provides mutual exclusion and a memory fence.

```go
var mu sync.Mutex
var counter int

func increment() {
    mu.Lock()
    counter++       // inside critical section
    mu.Unlock()     // happens-before the next Lock
}

func read() int {
    mu.Lock()
    defer mu.Unlock()
    return counter  // sees all prior increments
}
```

**Happens-before via sync/atomic**

Atomic operations establish ordering when used with the right memory guarantees. `atomic.StoreInt64` and `atomic.LoadInt64` synchronize on a specific variable. The Go memory model treats them as a sequentially consistent order if you stick to atomics exclusively, but mixing atomics with non-atomics requires care.

```go
var ready int32
var message string

func writer() {
    message = "world"
    atomic.StoreInt32(&ready, 1) // happens-before any load that observes 1
}

func reader() {
    for atomic.LoadInt32(&ready) == 0 {
    }
    fmt.Println(message) // guaranteed to print "world"
}
```

These constructs are the building blocks. The rules are formalized in the Go specification’s “Memory Model” section, which is concise but precise. We’ll unpack the machinery that enforces them.

---

### 2. Under the Hood

Two distinct subsystems interact here: the **goroutine scheduler** and the **memory model implementation**. Understanding both is essential for reasoning about performance and correctness.

#### 2.1 Scheduler Internals (G, M, P)

Go uses an M:N scheduler: it multiplexes *M* goroutines onto *N* OS threads. The core entities are:

- **G**: a goroutine. Contains a stack, the instruction pointer, and scheduling state.
- **M**: an OS thread. Executes goroutines.
- **P**: a processor (logical). Holds the run queue of goroutines waiting to execute and is required for an M to run Go code.

The number of P’s is set by `GOMAXPROCS`, defaulting to the number of CPUs. Each P maintains a local run queue, usually a circular buffer of up to 256 goroutines. When an M is assigned a P, it repeatedly pops a G from that P’s run queue and executes it. If the local queue is empty, the M will try to **steal** work from another P’s queue (work stealing). The global run queue serves as an overflow when local queues are full.

**Preemption and sysmon**

Goroutines are preemptively scheduled. Since Go 1.14, asynchronous preemption is based on signals. A dedicated OS thread called `sysmon` runs periodically and checks for goroutines that have been executing for too long (around 10 ms) or are blocking on system calls. If a goroutine is stuck in a tight loop without calling functions, `sysmon` sends a `SIGURG` signal to the running thread. The signal handler sets a flag that causes the goroutine to yield at a safe point (usually a function prologue or a loop back-edge). This prevents a single CPU-bound goroutine from starving others. Preemptive scheduling is critical to the memory model because it ensures that synchronisation primitives actually release control; without preemption, a spinning goroutine could block the thread indefinitely, preventing other goroutines from observing its state changes.

**Goroutine state transitions**

A G moves between states: `_Gidle`, `_Grunnable`, `_Grunning`, `_Gsyscall`, `_Gwaiting`, `_Gdead`. When a goroutine blocks on a channel or a lock, it is placed in a wait state and the associated object (channel, mutex) maintains a queue of waiting goroutines. Once the blocking condition is resolved, the goroutine is marked runnable and placed back on a run queue. This cooperative mechanism interacts with the memory model: the goroutine that unblocks another is responsible for ensuring that any stores it performed are visible to the waking goroutine.

#### 2.2 Stack Growth

Every goroutine starts with a small stack (typically 2 KB). The runtime transparently grows and shrinks the stack as needed. Early Go used **segmented stacks**, but modern Go (since 1.3) uses **contiguous stacks** with copying.

When the compiler detects that a function might need more stack, it inserts a stack growth preamble. At function entry, a comparison checks whether the stack pointer is below `stackguard0`. If so, the runtime allocates a new, larger stack (usually double the size), copies the entire old stack and all live pointers to the new one, adjusts pointers, and then resumes the goroutine on the new stack. This copying requires accurate pointer maps—a strong reason Go uses a precise, non-moving GC and stack frame layout. Stack shrinking happens during GC; if the goroutine’s used stack space is less than one quarter of its current stack, the runtime may shrink it back down.

Why does this matter for the memory model? Stack copies are transparent at the language level, but they can move pointers. Any code that holds pointers into a goroutine’s own stack (e.g., via `unsafe` tricks) must pin the goroutine’s stack or risk corruption. In normal, idiomatic Go you never notice; the compiler guarantees that all references to stack-allocated objects are updated correctly.

#### 2.3 Memory Model Implementation

The memory model is described in terms of *happens-before* relations, a partial order that determines visibility. The runtime enforces these relations via low-level memory barriers and atomic instructions appropriate to the CPU architecture.

For channel operations, `runtime.chansend` and `runtime.chanrecv` use memory barriers. When a send happens-before a receive, the send acquires a lock on the channel’s internal mutex, writes the element, and releases the lock. The receive acquires the same lock, reads the element, and then releases it. Because `Unlock` in Go acts as a release-acquire barrier (the exact semantics are those of `runtime.mutex` using `futex` on Linux, with appropriate atomic store/load), all writes done before the send are visible to the receive.

For `sync.Mutex`, the implementation uses `runtime.semacquire` and `runtime.semrelease`, which ultimately depend on atomic compare-and-swap and the OS’s futex. Go’s memory model guarantees that an unlock synchronises with the next lock that observes the unlocked state.

Atomic operations in the `sync/atomic` package follow sequential consistency by default, meaning they act as full memory barriers. However, Go 1.19 introduced the `atomic.Int64` and similar types that allow relaxed atomic operations only if you explicitly use `Store` and `Load`, which still act as sequentially consistent atomics. There are no C++-style acquire/release atomics in the standard API; this keeps the memory model simple.

The Go specification’s memory model is intentionally short and rule-based: “a read of a variable *r* is allowed to observe a write to *w* if *w* is visible to *r*.” Visibility is defined by a set of synchronisation operations. Anything not covered by these rules results in a *data race*, and the behaviour is undefined. The race detector instruments the runtime to catch such cases.

---

### 3. Why This Design?

Go’s designers made deliberate trade-offs to keep concurrent programming understandable and avoid the complexity found in some other languages.

**Why M:N scheduling instead of 1:1 threads?**
A 1:1 threading model (e.g., Java’s early green threads, or plain pthreads) makes each user-level thread a kernel thread. That scales poorly for hundreds of thousands of connections because kernel threads consume significant memory and scheduling overhead. Go’s M:N scheduler lets you launch millions of goroutines by multiplexing them onto a handful of OS threads. The choice of work stealing with per-P run queues is inspired by the Erlang VM and the Cilk runtime; it scales well with CPU count and avoids global lock contention.

**Why contiguous stacks with copying instead of segmented stacks?**
Segmented stacks (used in early Go) allowed stack extension by linking new segments, but they suffered from “hot split” problems: if a tight loop crossed a segment boundary repeatedly, performance tanked. Contiguous copying eliminates this issue at the cost of occasional copying overhead. The trade-off is usually worthwhile because most goroutines have small stacks that rarely grow.

**Why a simple happens-before model instead of Java’s JMM or C++ atomics?**
The Go memory model is a minimal set of rules. There are no volatile variables, no acquire/release ordering knobs, and no complex causality loops. The philosophy: “concurrent programming is hard enough; the language should not make it harder with a memory model that requires deep study of memory orders.” By linking synchronisation to only a handful of standard constructs (channels, mutexes, atomics), Go reduces the cognitive load. The cost is that developers cannot micro-optimise with relaxed atomics; but in practice, the simplicity wins because the majority of code is correct out of the box. This aligns with Go’s pillar *Simplicity over Complexity*.

**Why not a formal actor model like Erlang?**
Go shares memory between goroutines by default. This is a pragmatic decision: shared memory is often more natural for certain stateful workloads, and channels provide communication. The motto “Share memory by communicating” encourages a style, but Go does not enforce it—you can use mutexes and atomics just as easily. The design gives programmers the freedom to choose the best tool for the job while providing clear rules about when communication is safe.

---

### 4. Competing Approaches

How does Go’s runtime and memory model compare with other ecosystems?

**Java (JVM)**
Java’s memory model (JMM) is highly formalised, defining *happens-before* relationships across `volatile`, `synchronized`, `java.util.concurrent` latches, and atomic variables with various memory orderings. The JVM’s JIT compilation can perform aggressive optimisations that the JMM must constrain. Java threads are 1:1 kernel threads; the JVM may employ lightweight threads (Project Loom), but the memory model remains complex. Go’s model is simpler, with fewer concepts.

**C++ (and C)**
C++ provides a detailed atomics library with six memory orders (`memory_order_relaxed`, `acquire`, `release`, `seq_cst`, etc.). This gives expert programmers fine control over hardware barriers, but it’s extremely easy to introduce subtle bugs. Go omits this entirely; `sync/atomic` provides only sequentially consistent operations. C++ threads are 1:1. Go’s scheduler avoids the heavyweight creation and context-switch costs of C++ threads.

**Rust**
Rust’s memory model for atomics mirrors C++ (similar memory orders). Its ownership system, however, eliminates data races at compile time for non-atomic data. Rust’s async runtimes (e.g., Tokio) implement work-stealing schedulers similar to Go’s, but they are library-level, not part of the language runtime. Go bakes the scheduler into the runtime, giving a uniform experience without needing to pick an executor. The memory model is conceptually smaller.

**Erlang**
Erlang uses the actor model with no shared memory (message passing only). This eliminates data races completely. Go’s memory model applies to shared memory, so races are possible. Erlang’s scheduler is also work-stealing but preempts by counting reductions (function calls). Go’s model is more flexible for performance-critical state that benefits from mutex-based sharing.

**Node.js (JavaScript)**
Node.js is single-threaded with an event loop; no true parallel memory access exists in a single process (ignoring Worker Threads). This eliminates the need for a complex memory model but limits CPU utilisation. Go provides real parallelism via multiple P’s, so a memory model is necessary.

---

### 5. Common Mistakes

The most dangerous mistakes violate the happens-before guarantees, resulting in data races that may pass tests but fail in production.

**Mistake 1: Assuming goroutine start synchronises memory**

Spawning a goroutine with `go f()` does **not** create a happens-before edge. Any writes before the `go` statement may not be visible inside the new goroutine unless you use explicit synchronisation (like a channel or a `sync.WaitGroup`).

```go
var data string

// BAD: no synchronisation; data may not be visible.
func start() {
    data = "important"
    go func() {
        fmt.Println(data) // possible empty string
    }()
}
```

**Fix**: pass data via channel or use a lock.

**Mistake 2: Unsynchronised map access**

Concurrent reads and writes on a plain map cause a fatal runtime panic. The race detector catches this, but without it, the program can crash unpredictably.

```go
var m = make(map[string]int)
go func() { m["x"] = 1 }()
fmt.Println(m["x"]) // data race, may panic
```

**Fix**: protect with a mutex or use `sync.Map`.

**Mistake 3: Relying on goroutine ordering without channels**

Multiple goroutines operating on shared state without synchronisation have undefined visibility. Even if you see “works on my machine,” the compiler and CPU may reorder instructions.

```go
var a, b int
go func() { a = 1; b = 2 }()
go func() { fmt.Println(b, a) }() // could print 0,0 or 2,0 or 0,1
```

**Fix**: use atomic stores/loads or a channel.

**Mistake 4: Incorrect use of `sync/atomic` with non-atomic variables**

Atomic operations only guarantee ordering with respect to other atomic operations on the same variable. They do not act as a fence for regular reads/writes unless the happens-before rule is satisfied via the atomic operation itself. For example, writing a regular variable after an atomic store does not make that write visible to a goroutine that observes the store; you must arrange the order so the write happens before the store.

```go
var ready int32
var msg string
// Goroutine A:
msg = "hello"
atomic.StoreInt32(&ready, 1) // OK, store after msg

// Goroutine B:
if atomic.LoadInt32(&ready) == 1 {
    fmt.Println(msg) // guaranteed to see msg
}
```
If you swapped the order in A, the message might not be visible. The pattern must preserve write-before-atomic-store and read-after-atomic-load.

**Mistake 5: Leaking goroutines and holding locks while blocked**

Blocking in a critical section (e.g., I/O, channel operation) while holding a mutex can stall other goroutines. Go’s scheduler cannot preempt a goroutine holding a lock if that goroutine is blocked on a channel—it remains asleep, keeping the lock. This leads to stalls and unintended ordering reversals.

**Mistake 6: Missing stack growth in recursion**

While stack growth is automatic, extremely deep recursion can still exhaust the OS memory because each new stack copy allocates heap memory. The runtime sets a max stack size (1 GB by default on 64-bit), after which it panics. Understanding stack boundaries helps in writing recursive algorithms that don’t unexpectedly crash.

---

### 6. Performance Considerations

**Scheduling overhead**

Work-stealing is fast, but contention on the global run queue can occur when many goroutines are spawned and local queues overflow. The cost of a goroutine switch is on the order of tens of nanoseconds, far cheaper than an OS context switch (microseconds). Still, spawning millions of short-lived goroutines can cause measurable work-stealing overhead. Pooling goroutines or using a worker pool pattern can amortise this cost.

**Stack growth**

Copying a goroutine’s stack during growth involves memory allocation and pointer adjustment. If a goroutine grows its stack frequently (e.g., through recursive calls that breach the guard page repeatedly), it can incur noticeable overhead. Pre-allocating a larger stack is not directly possible, but you can limit deep recursion or allocate objects on the heap to keep frames small.

**Lock contention and cache-line bouncing**

A heavily contended `sync.Mutex` causes goroutines to sleep and wake, which involves futex calls. Profiling will show time spent in `runtime.lock2` or `sync.Mutex.Lock`. Reducing critical section duration, sharding data structures, or using atomics for simple counters can help. False sharing—when two unrelated variables share a cache line and are updated by different goroutines—degrades performance. Using padding (e.g., `_ [64]byte`) is sometimes necessary for high-throughput data structures.

**Channel vs. mutex/atomic**

Unbuffered channels provide strong synchronisation but incur the overhead of context switching if the partner goroutine is not ready. Buffered channels act like in-memory queues and can be more efficient for producer-consumer pipelines, but they still require internal locking. When only a single flag or counter needs protection, atomics are the cheapest; they compile to a single atomic instruction (e.g., `LOCK XADD` on x86). The Go memory model’s sequential consistency for atomics adds a memory barrier, which is slightly heavier than relaxed atomics in C++, but still lighter than a mutex.

**GC interaction**

Goroutine stacks are roots for the garbage collector. Deep stacks with many live pointers increase root scanning time. Large numbers of idle goroutines (leaks) consume memory that the GC must trace, raising pause times. The scheduler cooperates with the GC during mark termination: all goroutines must reach a safe point. Having too many active goroutines can increase GC pause latency.

---

### 7. Best Practices

**1. Use the race detector religiously**
`go test -race`, `go run -race`, and even `go build -race` for production binaries (with caution due to overhead). The detector instruments memory accesses and synchronisation primitives to detect violations of the memory model. It adds ~10x CPU overhead but catches races that are otherwise nearly impossible to diagnose.

**2. Keep synchronisation explicit and small**
Channel usage should transfer ownership: “here is the data, I no longer need it.” Mutexes should guard minimal state. Avoid holding locks across function calls that may block. This reduces deadlock risk and makes happens-before reasoning local.

**3. Prefer `sync.WaitGroup` for goroutine completion**
A `WaitGroup` provides the necessary happens-before edge: `wg.Add` happens-before the goroutine starts (conceptually), and `wg.Done` happens-before `wg.Wait` returns. This ensures all side effects of finished goroutines are visible.

**4. Document goroutine ownership and lifetime**
Each goroutine should have a clear responsibility and a defined point where it terminates. Avoid “fire-and-forget” goroutines without a cancellation mechanism. Use `context.Context` to propagate deadlines and cancellation.

**5. Use atomics for simple state flags**
For a “done” flag or a counter used solely for statistics, `atomic.Bool` or `atomic.Int64` is the most lightweight option and establishes clear ordering when used correctly. It avoids the overhead of a mutex while maintaining correctness.

**6. Avoid assuming a particular interleaving**
Write tests that stress concurrent code with many goroutines and varying `GOMAXPROCS` to surface memory-model bugs. Tools like `stress` or custom test harnesses can help. Remember: data races are undefined behaviour; any output is allowed.

---

### 8. Examples

**Example 1: Detecting a data race with the race detector**
Save the following as `race_example_test.go` and run `go test -race`.

```go
package main

import (
    "fmt"
    "testing"
)

func TestRace(t *testing.T) {
    counter := 0
    done := make(chan bool)

    go func() {
        counter++ // write without synchronisation
        done <- true
    }()

    counter++ // concurrent write
    <-done
    fmt.Println(counter)
}
```

The race detector will report two goroutines unsynchronised write to `counter`.

**Example 2: A thread-safe cache using a mutex**
This pattern respects the memory model because all reads and writes are done under the same lock.

```go
type Cache struct {
    mu   sync.RWMutex
    data map[string]string
}

func NewCache() *Cache {
    return &Cache{data: make(map[string]string)}
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    v, ok := c.data[key]
    return v, ok
}

func (c *Cache) Set(key, value string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.data[key] = value
}
```

An RWMutex allows multiple concurrent readers; a write lock happens-before all subsequent read locks.

**Example 3: Goroutine coordination with `sync.WaitGroup`**
Each worker writes to a slice, and the `WaitGroup` ensures all writes are visible to the main goroutine after `Wait` returns.

```go
func main() {
    var wg sync.WaitGroup
    results := make([]int, 5)
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(idx int) {
            defer wg.Done()
            results[idx] = idx * 2 // safe: no concurrent writes to same index
        }(i)
    }
    wg.Wait()
    fmt.Println(results)
}
```

Because `wg.Done` synchronises with `wg.Wait`, the writes are visible.

---

### 9. Summary & Exercises

**Summary**
The Go runtime orchestrates concurrency through an M:N work-stealing scheduler and contiguous, dynamically growing stacks. The memory model defines a set of happens-before relations anchored by synchronisation primitives: channel operations, mutex locks/unlocks, and atomic operations. This design keeps the model simple enough to reason about while giving performance-conscious developers the tools to build scalable systems. The scheduler’s preemption and work-stealing ensure fairness and responsiveness. Memory visibility is guaranteed only when you follow the synchronisation rules; any deviation is a data race with undefined consequences. Profiling and the race detector are indispensable allies in writing correct, efficient concurrent Go programs.

**Exercises**

1. **Build a thread-safe LRU cache with TTL**
   Implement a concurrent LRU cache that supports `Get`, `Put`, and `Delete` operations. Use a mutex to synchronise access. Add per-entry TTL expiration using a background goroutine that periodically sweeps expired items. Ensure that all operations are safe under the memory model and test under `-race`. Consider how to minimise lock contention while maintaining correctness.

2. **Diagnose and fix a race in a payment processing pipeline**
   You are given a simplified order-processing service: multiple goroutines update a shared `map[orderID]*Order` with status changes, while another goroutine periodically reads the map to send confirmation emails. The code has no synchronisation. Reproduce the race using the race detector, then refactor the code to be correct using one of: channels (pipeline), a mutex, or atomics. Justify your choice in terms of the memory model and throughput.

3. **Design a work-stealing dispatcher**
   Build a simple task scheduler where worker goroutines steal tasks from a shared queue. Use a lock-free queue based on `sync/atomic` for performance (e.g., a singly-linked list with atomic head/tail). Ensure that the memory ordering rules are satisfied: when a worker steals a task, it must see all writes performed by the producer. Write a stress test that verifies correctness under high concurrency, and profile the overhead compared to a channel-based dispatcher.

```
