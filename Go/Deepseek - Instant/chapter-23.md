# Chapter 23: Channels & Communication

If goroutines are Go’s units of concurrent execution, **channels** are the conduits that connect them. The slogan “Share memory by communicating, don’t communicate by sharing memory” is embodied in this single type. Channels are not just thread‑safe queues; they are a fundamental control structure that can synchronise, coordinate, and transfer ownership of data between goroutines without low‑level locks.

This chapter dissects channels from their basic syntax down to the runtime representation. You will learn how to use them safely, why they are designed the way they are, and when a channel is the wrong tool for the job.

---

## 1. Basic Usage

A channel is a typed conduit. You create it with `make`, send values with `<-`, and receive values with `<-` as an expression.

### Unbuffered (Synchronous) Channels

```go
// Create an unbuffered channel of integers.
ch := make(chan int)

// Send in one goroutine, receive in another.
go func() {
    ch <- 42 // blocks until someone receives
}()

value := <-ch // blocks until someone sends
```

### Buffered (Asynchronous) Channels

```go
// Create a buffered channel with capacity 3.
ch := make(chan string, 3)

ch <- "a" // does not block
ch <- "b"
ch <- "c"
// ch <- "d" // would block (buffer full)

fmt.Println(<-ch) // "a"
fmt.Println(<-ch) // "b"
```

### Closing and Ranging

A sender can close a channel to indicate that no more values will be sent.

```go
ch := make(chan int)

go func() {
    for i := 0; i < 5; i++ {
        ch <- i
    }
    close(ch)
}()

// The loop receives values until the channel is closed.
for v := range ch {
    fmt.Println(v)
}
```

### Directional Channels

You can restrict a channel to send‑only or receive‑only. This is a compile‑time constraint that clarifies intent.

```go
func producer(out chan<- int) {   // send‑only
    for i := 0; i < 10; i++ {
        out <- i
    }
    close(out)
}

func consumer(in <-chan int) {    // receive‑only
    for v := range in {
        fmt.Println(v)
    }
}

func main() {
    ch := make(chan int)
    go producer(ch)
    consumer(ch)
}
```

---

## 2. Under the Hood

The runtime representation of a channel is the `hchan` struct (defined in `runtime/chan.go`). Understanding it clarifies blocking behaviour, memory layout, and performance.

### The `hchan` Struct (Simplified)

```go
type hchan struct {
    qcount   uint           // total data in the queue
    dataqsiz uint           // size of the circular buffer
    buf      unsafe.Pointer // pointer to the buffer array
    elemsize uint16
    closed   uint32
    elemtype *_type
    sendx    uint           // send index in buffer
    recvx    uint           // receive index in buffer
    recvq    waitq          // list of blocked receivers
    sendq    waitq          // list of blocked senders
    lock     mutex
}
```

- **Lock:** A single mutex protects all fields. Channel operations are not lock‑free, but the lock is held for very short critical sections.
- **Circular buffer:** The ring buffer (`buf`) is allocated only when `dataqsiz > 0` (buffered channel). The buffer allows senders to proceed without a waiting receiver.
- **Wait queues:** `recvq` and `sendq` hold goroutines that are blocked. Each entry is a `sudog` (a runtime structure representing a blocked goroutine). When a value arrives, the runtime directly hands the value from the sender’s stack to the receiver’s stack if possible, avoiding an extra copy through the buffer.

### Blocking and Unblocking

1. **Send on an unbuffered channel:** The sender’s goroutine is parked (added to `sendq`) and its stack is stored. When a receiver appears, the runtime copies the value directly from the sender’s stack to the receiver’s stack and then wakes the sender.
2. **Receive on an unbuffered channel:** Similarly, the receiver parks on `recvq`. When a sender arrives, the value is copied directly.
3. **Buffered channel with space:** The sender copies the value into the ring buffer at position `sendx`, increments `qcount`, and continues without blocking.
4. **Buffered channel when empty:** The receiver parks on `recvq` until at least one element is in the buffer.
5. **Close:** The runtime sets `closed = 1`, then iterates over `recvq` and `sendq` to unblock all goroutines. Receivers get the zero value, senders panic.

### Memory Allocations

- `make(chan T)`: One allocation for the `hchan` struct.
- `make(chan T, N)`: One allocation for `hchan` plus a second allocation for the ring buffer (`buf`).
- Sending a value that does **not** escape to the heap? The value is copied either into the ring buffer (buffered) or directly to another goroutine’s stack (unbuffered). The channel itself never holds references that force heap allocation – but if the value contains pointers, those pointers obviously point to heap objects.

---

## 3. Why This Design?

### Channels as First‑Class CSP

Go’s concurrency is a direct implementation of **Communicating Sequential Processes (CSP)**, a formal model by Tony Hoare (1978). CSP treats processes (goroutines) as independent entities that communicate only through synchronous channels. No shared memory, no locks – processes are isolated except for the messages they exchange.

The Go team chose CSP over:
- **Actor model (Erlang, Akka):** Actors have mailboxes and addresses; communication is asynchronous. Go’s channels are typed and synchronous by default, which makes data flow and backpressure explicit.
- **Futures/promises (JavaScript, C++):** These are one‑shot, often heap‑allocated. Channels are reusable, support multiple producers/consumers, and integrate with `select`.

### Why Unbuffered by Default?

Unbuffered channels enforce a **rendezvous** – the sender and receiver meet at the same moment. This is the purest form of CSP: communication is the synchronisation. Unbuffered channels make the coordination point explicit and push developers toward clear handoffs.

Buffered channels are an optimisation, not the default. They decouple send and receive in time, but at the cost of introducing hidden state (the buffer). Over‑buffering can hide bugs like deadlocks and make reasoning about program correctness harder.

### Why Closing?

Closing is a signal: “no more values will be sent”. This turns a channel into a readable stream. Without a close, a receiver cannot distinguish between “there will be more later” and “done”. The `for range` over a channel depends on close to terminate.

Alternatives considered:
- **Sentinel value (e.g., `nil`):** Pollutes the data type, requires agreement on the sentinel, and cannot signal “no more” if `nil` is a valid value.
- **Separate `done` channel:** Still common for cancellation, but closing directly is simpler when you own the channel.

---

## 4. Competing Approaches

| Language / Library | Mechanism | Synchronisation | Type Safety | Notes |
|-------------------|-----------|----------------|-------------|-------|
| **Go** | Channels | Sync (unbuf) / Async (buf) | Yes (compile‑time) | Part of the language, not a library. `select` for multiplexing. |
| **Java** | `java.util.concurrent.BlockingQueue` | Async (queue) | Yes (generics) | Library interface, many implementations (`ArrayBlockingQueue`, `LinkedBlockingQueue`). No `select` equivalent – you use `poll` with timeouts or `put`. |
| **C++** | `std::condition_variable` + mutex + queue | Manual | No | You build the channel yourself. Error‑prone, no built‑in close semantics. |
| **C++ (actor)** | SObjectizer, CAF | Async | Yes (templates) | Heavy frameworks. Not part of the standard library. |
| **Rust** | `std::sync::mpsc` (multi‑producer, single‑consumer) | Async (buffer) | Yes (type system) | Zero‑cost abstractions. The `select!` macro is library‑based, less ergonomic than Go’s `select`. |
| **Python** | `queue.Queue` | Async (buffer) | No (dynamic) | GIL limits true parallelism. `asyncio.Queue` for async/await. |
| **Erlang/Elixir** | Process mailboxes | Async | Yes (dynamic types) | Each process has a mailbox; `receive` selects messages. No typed channels. |

### Key Distillation

- **Java/C#** prioritise library solutions with many knobs (capacity policies, fairness, timeouts). Go hard‑codes the CSP model into the syntax, making communication patterns trivially readable.
- **Rust** achieves zero‑cost with its ownership system, but the single‑consumer restriction (`mpsc`) is a conscious design trade‑off. Go’s channels allow any number of producers and consumers on the same channel.
- **Python’s `queue.Queue`** is blocking but suffers from the GIL – it’s fine for I/O‑bound work but cannot parallelise CPU‑bound tasks. Go’s channels drive real parallelism.

---

## 5. Common Mistakes

### 1. Goroutine Leak from an Unclosed Channel

```go
func leak() {
    ch := make(chan int)
    go func() {
        v := <-ch // waits forever
        fmt.Println(v)
    }()
    // ch is never closed, and nothing sends on it.
    // The goroutine stays blocked indefinitely.
}
```

**Fix:** Either close the channel (if you are the sender) or ensure a send eventually occurs.

### 2. Sending on a Closed Channel (Panic)

```go
ch := make(chan int)
close(ch)
ch <- 42 // panic: send on closed channel
```

**Fix:** Only the goroutine that owns the channel should close it. Use `sync.Once` or a guard boolean if multiple goroutines might attempt a close.

### 3. Deadlock from Improper Order

```go
func main() {
    ch := make(chan int)
    ch <- 1 // blocks forever because no receiver is ready
    v := <-ch
    fmt.Println(v)
}
```

**Fix:** Launch the receiver before the send, or use a buffered channel.

### 4. Using a Buffered Channel as a “Queue” Without Backpressure

```go
ch := make(chan Work, 1000000) // huge buffer
for {
    w := getWork()
    ch <- w // never blocks, hides producer/consumer imbalance
}
```

A large buffer masks problems until memory overflows. Buffered channels should have a bounded capacity that reflects your tolerance for latency vs. throughput.

### 5. `for range` on a Channel That Is Never Closed

```go
func produce(ch chan int) {
    for i := 0; i < 10; i++ {
        ch <- i
    }
    // missing close(ch)
}

func main() {
    ch := make(chan int)
    go produce(ch)
    for v := range ch { // deadlock after 10 values, because channel still open
        fmt.Println(v)
    }
}
```

**Fix:** Always close the channel from the sending side when you know no more values will be sent.

---

## 6. Performance Considerations

### Cost of a Channel Operation

| Operation | Approximate cost (relative to mutex lock/unlock) |
|-----------|--------------------------------------------------|
| Unbuffered send/receive (no buffer) | ~10× a mutex (because of goroutine parking / unparking) |
| Buffered send (buffer not full) | ~1.5× a mutex |
| Buffered receive (buffer not empty) | ~1.5× a mutex |
| Contended channel | Degrades non‑linearly as the wait queues grow |

- **Memory barrier:** Channels always imply a *happens‑before* relationship. The runtime issues a memory barrier on send/receive, which incurs a CPU cost.
- **Contention:** A single channel shared by many goroutines becomes a bottleneck because all operations serialise on the `hchan.lock`. For high‑contention scenarios, consider sharding (multiple channels) or use a specialised structure like a lock‑free ring buffer.

### When to Prefer a Mutex

Channels are expressive but not always cheap. A `sync.Mutex` protecting a slice or map often outperforms a channel when you need to:

- **Hold a shared state for multiple reads/writes** (a channel would require copying the entire state per message).
- **Implement a long‑lived counter** – an atomic operation beats a channel round‑trip.
- **Avoid goroutine scheduling overhead** – a mutex that is rarely contended is extremely fast (user‑space spinning before falling back).

### Performance‑Sensitive Patterns

1. **Use buffered channels to reduce goroutine blocking** when you have predictable bursts. Tune the buffer size to the average load.
2. **Avoid sending pointers** if the underlying object never changes – but copying small structs (≤ 2 machine words) is cheap.
3. **Nil channels in `select`** – a `select` case on a `nil` channel is never selected. This is a deliberate optimisation to toggle behaviour at zero cost.

---

## 7. Best Practices

### 1. Channel Ownership

A goroutine that creates a channel owns it. The owner:
- Decides whether the channel is buffered or not.
- Is responsible for closing it (if necessary).
- May pass **send‑only** or **receive‑only** views to other goroutines.

```go
func owner() {
    ch := make(chan int, 10)
    defer close(ch) // owner closes
    go worker(ch)   // gives worker a receive‑only view
    for i := 0; i < 5; i++ {
        ch <- i
    }
}

func worker(ch <-chan int) { // receive‑only ensures worker cannot close
    for v := range ch {
        fmt.Println(v)
    }
}
```

### 2. Use Directional Channels in Function Signatures

Even if you are not the owner, annotating the direction prevents bugs at compile time.

```go
func sendOnly(out chan<- int) { ... }
func recvOnly(in <-chan int) { ... }
```

### 3. Never Close from the Receiver Side

If a receiver closes the channel while a sender is still trying to write, the sender panics. Only the goroutine that knows the complete lifecycle should call `close`.

### 4. Favour Unbuffered Channels for Synchronisation

When you need a signalling mechanism (e.g., “I am done”), use an unbuffered channel of an empty struct:

```go
done := make(chan struct{})
go func() {
    doWork()
    close(done) // signal
}()
<-done // wait
```

`struct{}` occupies zero bytes, making the intention clear.

### 5. Use `select` with Default for Non‑Blocking Operations

```go
select {
case v := <-ch:
    fmt.Println(v)
default:
    fmt.Println("no value ready")
}
```

But be careful: busy loops with `default` can consume 100% CPU. Use timeouts or backoff instead.

---

## 8. Examples

### Example 1: Worker Pool with Channels

A simple, idiomatic worker pool. The `Work` type can be any struct.

```go
package main

import (
    "fmt"
    "sync"
)

func worker(id int, jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    for job := range jobs {
        results <- job * 2 // simulate processing
    }
}

func main() {
    const numJobs = 100
    const numWorkers = 5

    jobs := make(chan int, numJobs)
    results := make(chan int, numJobs)

    var wg sync.WaitGroup
    for w := 0; w < numWorkers; w++ {
        wg.Add(1)
        go worker(w, jobs, results, &wg)
    }

    // Send jobs
    for j := 0; j < numJobs; j++ {
        jobs <- j
    }
    close(jobs)

    // Wait for workers to finish, then close results
    go func() {
        wg.Wait()
        close(results)
    }()

    // Collect results
    for r := range results {
        fmt.Println(r)
    }
}
```

### Example 2: Pipeline with Graceful Cancellation

A three‑stage pipeline (generate → square → print) that can be cancelled via a `done` channel.

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func generate(ctx context.Context, numbers ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range numbers {
            select {
            case out <- n:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func square(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case out <- n * n:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
    defer cancel()

    pipeline := square(ctx, generate(ctx, 1, 2, 3, 4, 5))

    for v := range pipeline {
        fmt.Println(v)
    }
}
```

### Example 3: Non‑Blocking Send with Select

Sometimes you want to try a send, but fall back to another action if the channel is full.

```go
func trySend(ch chan<- Work, w Work, timeout time.Duration) error {
    select {
    case ch <- w:
        return nil
    case <-time.After(timeout):
        return fmt.Errorf("send timed out after %v", timeout)
    }
}
```

---

## 9. Summary & Exercises

### Summary

- **Channels** are typed, concurrent‑safe conduits that embody the “share memory by communicating” principle.
- **Unbuffered channels** synchronise sender and receiver; **buffered channels** decouple them in time at the cost of hidden state.
- The runtime uses a mutex‑protected ring buffer with wait queues for blocked goroutines.
- Go’s design (CSP, first‑class channels) contrasts with library‑based queues in Java/C++ and actor mailboxes in Erlang.
- Common mistakes: goroutine leaks, deadlocks, closing from the receiver side, and unbounded buffering.
- Performance: channels are heavier than mutexes for simple shared state but indispensable for structured communication.
- Best practices: explicit ownership, directional types, `select` for timeouts, and `for range` for reading until close.

### Exercises

1. **Fan‑out / Fan‑in Pipeline**  
   Build a pipeline that reads a file line by line, fans out those lines to three processing goroutines (each transforms the line to uppercase), and fans in the results to a single output goroutine that writes to another file. Use channels and `sync.WaitGroup`. Ensure the pipeline shuts down cleanly even if one worker panics.

2. **Multiplexer with Priority**  
   Implement a function `func MergeWithPriority[T any](primary, secondary <-chan T) <-chan T` that returns a channel. It should always receive from `primary` if a value is immediately available; otherwise, it falls back to `secondary`. Never block waiting for `primary` if `secondary` has data. Use `select` with `default` or reflection (but try without `reflect` first). Write a benchmark comparing it against `Merge` that simply interleaves arbitrarily.

3. **Rate‑Limiting Proxy**  
   Write a rate‑limiting proxy that forwards requests from many goroutines to a slow API. The proxy must limit throughput to 10 requests per second. Use a ticker channel for the rate limiter and a buffered channel as a work queue. Include a cancellation mechanism so that `ctx.Done()` stops the proxy immediately. Measure the request latency under load.

---

**Chapter 23 complete.** In the next chapter, we explore the `select` statement – the multi‑way switch that makes channels truly composable. We will dissect its non‑deterministic nature, default cases, and how to build robust timeouts and cancellations.
