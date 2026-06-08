## Chapter 25: Advanced Concurrency Patterns

Go’s concurrency primitives—goroutines, channels, `select`, and `context`—compose into a rich set of patterns that solve real-world distributed and parallel problems with elegance. This chapter moves beyond basic channel usage into the design of pipelines, fan‑in/fan‑out topologies, worker pools, rate limiting, and the request‑scoped control that `context.Context` provides. Each pattern embodies the Go philosophy: **share memory by communicating**, not through locks and shared state.

### 1. Basic Usage

#### Pipelines

A pipeline is a series of stages connected by channels, where each stage is a group of goroutines that receive values from an upstream channel, perform some transformation, and send results downstream.

```go
// Stage 1: generator
func gen(ctx context.Context, nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            select {
            case out <- n:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

// Stage 2: squarer
func sq(ctx context.Context, in <-chan int) <-chan int {
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

// Usage: ctx, cancel := context.WithCancel(context.Background())
// defer cancel()
// for res := range sq(ctx, gen(ctx, 2, 3, 4)) { fmt.Println(res) }
```

#### Fan‑Out / Fan‑In

Fan‑out splits work from a single input channel across multiple identical worker goroutines. Fan‑in multiplexes multiple input channels into a single output channel.

```go
func fanOut(ctx context.Context, in <-chan int, workers int) []<-chan int {
    outs := make([]<-chan int, workers)
    for i := 0; i < workers; i++ {
        outs[i] = sq(ctx, in) // each sq runs in its own goroutine
    }
    return outs
}

func fanIn(ctx context.Context, channels ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    multiplex := func(c <-chan int) {
        defer wg.Done()
        for v := range c {
            select {
            case out <- v:
            case <-ctx.Done():
                return
            }
        }
    }

    wg.Add(len(channels))
    for _, c := range channels {
        go multiplex(c)
    }

    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}
```

#### Worker Pool

A worker pool limits concurrency by using a fixed number of goroutines that process jobs from a shared channel. Results are collected via a separate channel.

```go
type Job struct {
    ID   int
    Data string
}

type Result struct {
    JobID int
    Value string
}

func worker(ctx context.Context, jobs <-chan Job, results chan<- Result) {
    for job := range jobs {
        select {
        case results <- Result{JobID: job.ID, Value: strings.ToUpper(job.Data)}:
        case <-ctx.Done():
            return
        }
    }
}

func pool(ctx context.Context, numWorkers int) {
    jobs := make(chan Job, numWorkers)
    results := make(chan Result, numWorkers)

    for i := 0; i < numWorkers; i++ {
        go worker(ctx, jobs, results)
    }
    // enqueue jobs ...
    close(jobs) // signal workers that no more jobs are coming
    // collect results ...
}
```

#### Rate Limiting & Backpressure

Rate limiting can be achieved with a `time.Ticker` or a buffered channel acting as a token bucket. Backpressure is naturally built into unbuffered channels—the sender blocks until the receiver is ready.

```go
func rateLimited(ctx context.Context, requests <-chan string, limit time.Duration) {
    ticker := time.NewTicker(limit)
    defer ticker.Stop()
    for req := range requests {
        select {
        case <-ticker.C:
            // process req
        case <-ctx.Done():
            return
        }
    }
}

// Token bucket with burst capacity
type tokenBucket struct {
    tokens chan struct{}
}

func newTokenBucket(rate int, burst int) *tokenBucket {
    tb := &tokenBucket{
        tokens: make(chan struct{}, burst),
    }
    go func() {
        ticker := time.NewTicker(time.Second / time.Duration(rate))
        defer ticker.Stop()
        for range ticker.C {
            select {
            case tb.tokens <- struct{}{}:
            default: // bucket full, drop token
            }
        }
    }()
    return tb
}

func (tb *tokenBucket) Wait(ctx context.Context) error {
    select {
    case <-tb.tokens:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

#### Pub/Sub with Channels

A simple publish‑subscribe pattern can be built using a central hub that fans out messages to subscriber channels.

```go
type Broker struct {
    subscribers map[chan string]struct{}
    mu          sync.RWMutex
}

func (b *Broker) Subscribe() <-chan string {
    ch := make(chan string, 16)
    b.mu.Lock()
    b.subscribers[ch] = struct{}{}
    b.mu.Unlock()
    return ch
}

func (b *Broker) Unsubscribe(ch <-chan string) {
    b.mu.Lock()
    delete(b.subscribers, ch.(chan string))
    b.mu.Unlock()
}

func (b *Broker) Publish(ctx context.Context, msg string) {
    b.mu.RLock()
    defer b.mu.RUnlock()
    for ch := range b.subscribers {
        select {
        case ch <- msg:
        case <-ctx.Done():
            return
        default: // non‑blocking send; drop if subscriber is slow
        }
    }
}
```

#### Context: Cancellation, Deadlines, Scoped Values

`context.Context` provides cancellation propagation and deadline enforcement across goroutine boundaries, plus request‑scoped key‑value storage.

```go
// Cancellation with cause (Go 1.20+)
ctx, cancel := context.WithCancelCause(context.Background())
go func() {
    time.Sleep(2 * time.Second)
    cancel(fmt.Errorf("request timed out"))
}()
select {
case <-ctx.Done():
    log.Println(context.Cause(ctx)) // request timed out
}

// Deadline
ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(100*time.Millisecond))
defer cancel()
// Pass ctx to network call; it will be cancelled after 100ms.

// Request‑scoped values (use sparingly)
ctx = context.WithValue(ctx, requestIDKey, "req-1234")
// Retrieve: ctx.Value(requestIDKey)
```

### 2. Under the Hood

Every advanced concurrency pattern ultimately runs on the Go runtime’s goroutine scheduler, which multiplexes goroutines onto OS threads using an **M:N model**. Here’s how the mechanics translate:

**Goroutine Lifecycle in Pipelines:**
When a pipeline stage is written as a goroutine with a `for range` over an input channel, the goroutine is blocked on channel receive. The scheduler deschedules it and puts it into a wait queue associated with that channel. When a sender pushes a value, the scheduler wakes the goroutine and places it on a run queue. Because goroutines are cheap (starting at a few KiB of stack), pipelines can have thousands of stages without overwhelming the OS.

**Channel Internals:**
A channel (`hchan` struct) holds a circular buffer, a mutex, and sender/receiver wait lists. Unbuffered channels synchronize directly: a sender blocks until a receiver is ready, or vice versa. In fan‑in patterns, multiple senders contend for the same channel mutex, but the overhead is minimal compared to thread‑safe queues in other languages. The `select` statement compiles into a runtime function that randomly chooses among ready cases, avoids starvation, and integrates with channel wait queues.

**Context Cancellation Propagation:**
A `context.Context` is a tree of nodes. `WithCancel` creates a child node with a `Done()` channel. When `cancel()` is called, it closes the child’s `Done` channel and recursively cancels all descendants. This closure triggers all `select` statements waiting on `<-ctx.Done()` without any explicit polling. The `Cause` mechanism (Go 1.20+) attaches an error to the cancellation, making debugging easier. The context also carries deadlines: the `WithDeadline` function sets a timer that calls `cancel` when the time arrives. This timer runs in a separate goroutine managed by the runtime’s timer heap, ensuring minimal overhead.

**Worker Pool Scheduling:**
A worker pool with `N` goroutines reading from a buffered channel creates `N` consumers that compete for jobs. The channel’s buffer size defines how much backpressure is absorbed before senders block. The scheduler’s work‑stealing algorithm distributes these goroutines across CPU cores. An optimal pool size is typically related to `GOMAXPROCS` (number of OS threads) and the I/O‑bound vs CPU‑bound nature of the work.

### 3. Why This Design?

Go’s concurrency patterns are built entirely out of goroutines, channels, and `select` because they deliver **composable concurrency** without intermediate abstractions.

**Pipelines vs. Thread‑Pool Executors**
In Java, a pipeline might be expressed using `CompletableFuture` chains or reactive streams, which introduce a framework‑specific vocabulary. Go’s pipelines are ordinary goroutines and channels. You can reason about data flow by reading code top‑to‑bottom, without understanding a library’s internal state machine. The lack of a thread‑pool abstraction is deliberate: goroutines are so cheap that you spawn one per stage; you don’t need to manage a thread pool. This eliminates whole classes of bugs related to thread starvation or pool sizing.

**Implicit Backpressure**
Unbuffered channels automatically apply backpressure—the sender cannot proceed until the receiver is ready. This is a form of **synchronous rendezvous**. It removes the need for explicit flow control or reactive pull models. The system naturally slows down if a downstream stage is slow, preventing unbounded memory growth. The choice of unbuffered vs. buffered channels gives the programmer a simple knob to tune throughput vs. latency.

**Context as a First‑Class Cancellation Scope**
Before `context.Context` was introduced, cancellation in Go relied on closing a `done` channel. That pattern is still valid, but `context` standardises timeouts, deadlines, and value propagation. The tree structure means a single cancellation can cleanly shut down an entire call graph. It’s more than a thread‑local variable; it’s an explicit parameter that makes the control flow visible, unlike the implicit thread interrupts or `CancellationToken` in .NET that can be passed opaquely.

**Channels as the Universal Connector**
Channels unify producer‑consumer, pub‑sub, and work‑queue patterns without requiring a heavyweight message broker. They are memory‑safe, typed, and integrate with `select` for multiplexing. This uniformity means developers learn one primitive and can build everything from simple pipelines to complex fan‑in structures. The philosophy is: **compose, don’t inherit**—combine small, well‑understood pieces into larger behaviours.

### 4. Competing Approaches

| Language / Framework | Mechanism | Comparison |
|----------------------|-----------|------------|
| Java (java.util.concurrent) | `ExecutorService`, `BlockingQueue`, `CompletableFuture` | Java separates thread management from tasks. A pipeline requires manual wiring of futures or reactive libraries. Go’s channels collapse queueing, synchronization, and scheduling into one simple abstraction. |
| Python (asyncio) | `asyncio.Queue`, coroutines, `gather` | asyncio coroutines are non‑preemptive and require an event loop. CPU‑bound work must be explicitly offloaded. Go goroutines are preempted by the runtime and truly parallel. Python’s `asyncio.Queue` offers similar backpressure, but lacks static typing and compile‑time safety. |
| C++ (std::thread, TBB) | Threads, `std::async`, concurrent queues | Manual thread management, no built‑in cancellation propagation. Context cancellation must be implemented with atomic flags. Go’s goroutines are far lighter, making fine‑grained pipelines feasible without pool management. |
| Erlang/Elixir | Actor model, message passing | The actor model is a higher‑level abstraction where each actor has a mailbox. Go’s channels are more flexible and don’t enforce the actor boundary. Both achieve isolation through message passing, but Erlang’s preemptive scheduling and OTP provide supervision trees, which Go lacks in the standard library. |
| Rust (async/await, tokio) | `tokio::sync::mpsc`, streams | Rust’s zero‑cost futures and channels give fine control over allocations. Cancellation is often tied to dropping a future. Go’s garbage collection and simpler ownership model trade some control for a vastly simpler programming experience. Both share the “channels as pipes” philosophy. |

### 5. Common Mistakes

**Goroutine Leaks in Pipelines**
If a pipeline stage exits early (due to an error, for example) without draining its input channel, upstream goroutines remain blocked forever trying to send. The solution is to always tie pipeline stages to a `context` that can be cancelled, or use a “done” channel. Even better, ensure that a stage that stops reading closes its input channel’s sender side, but that is often impossible if the sender is shared. Context cancellation is the safest universal lever.

**Closing Channels from the Receiver Side**
It’s a runtime panic. A receiver should never close a channel; only the sender knows when no more values will be sent. In fan‑in, the multiplexing goroutine is the only one that should close the merged output channel after all inputs have been drained and `sync.WaitGroup` completes.

**Ignoring `context.Context` Propagation**
When building a worker pool, forgetting to pass a context down to the workers means they cannot be cancelled when the main function returns. They become orphaned goroutines. Always thread `ctx` through every blocking operation (channel send, network call, time.Sleep alternative) and check `ctx.Err()`.

**Rate Limiter that Never Releases**
A common mistake is to create a `time.Ticker` inside a request handler without stopping it, causing a goroutine leak. Always `defer ticker.Stop()`. Another is using an unbuffered token bucket where the token producer blocks when the bucket is full, effectively halting the producer. Use a buffered channel or a `select` with a `default` to drop tokens when full.

**Context Value Key Collisions**
Using a bare `string` as a context key risks collision between different packages. Always define unexported custom types for keys, or at least use package‑specific strings that aren’t exported. Collisions can lead to security issues when middleware accidentally overwrites a caller’s identity.

**Unbounded Fan‑In Without Memory Bounds**
When fan‑in merges multiple channels into one, if the consumer is slower than the producers, buffered channels can fill up and then block producers. Without a backpressure mechanism, the system may deadlock or consume excessive memory. Always design fan‑in with a bounded buffer and consider dropping or throttling when full.

### 6. Performance Considerations

**Channel Overhead**
Each channel operation involves a mutex lock, potential goroutine parking/unparking, and, for buffered channels, a memory copy into the buffer. Sending a value on a channel is in the range of tens of nanoseconds in the uncontended case, but under heavy contention that can increase. For high‑throughput pipelines, consider using a slice‑based ring buffer protected by a mutex for the hot path, then exposing a channel interface at the boundary. However, that sacrifices the “share by communicating” model; only optimise after profiling.

**Goroutine Scheduling**
The runtime uses work‑stealing and local run queues. Creating thousands of goroutines is fine, but spawning a goroutine per tiny task can lead to scheduling overhead dominating the work. A worker pool with a fixed number of goroutines reusing tasks can be more efficient for CPU‑bound workloads. For I/O‑bound tasks, the scheduler’s network poller integrates with channels; a goroutine blocked on network I/O doesn’t occupy a thread, so the overhead is minimal.

**Select Statement Cost**
A `select` with many cases is implemented as a runtime function that shuffles the order to ensure fairness and then probes each channel for readiness. The cost scales linearly with the number of cases. For critical sections with dozens of channels, consider alternative designs, but in practice a handful of cases is negligible.

**Context’s Memory Cost**
Each `WithCancel` or `WithTimeout` allocates a new context node (~96 bytes) and a timer goroutine (for deadlines). Creating a context per request is standard and cheap; creating one per iteration in a tight loop might be wasteful. Reuse contexts when possible, or pass a parent context down.

**Token Bucket Allocation**
A token bucket implemented with a buffered channel and a ticker allocates a new goroutine for the ticker. For many rate limiters, the overhead is negligible. An alternative is a `rate.Limiter` from `golang.org/x/time/rate`, which is allocation‑friendly and uses a single background goroutine if needed. Measure before reaching for an external package; a simple `time.Ticker` might suffice.

**Backpressure and Buffer Sizing**
Buffered channels decouple producer and consumer, but too large a buffer hides bottlenecks and consumes memory. In a pipeline, measure end‑to‑end latency and throughput. A good starting point is a buffer size equal to the expected number of concurrent producers. Remember that a full buffer still blocks the sender; that’s your backpressure indicator.

### 7. Best Practices

1. **Make Context the First Parameter**
   Every function that performs I/O or blocks should accept `context.Context` as its first argument, named `ctx`. This is idiomatic and expected by the entire ecosystem.
   ```go
   func fetchURL(ctx context.Context, url string) ([]byte, error) { ... }
   ```

2. **Close Channels Only from the Sender**
   In a pipeline, the final stage often receives from a channel until it’s closed. The sender (usually the producer goroutine) must close the channel when it has finished. Use `defer close(ch)` inside the goroutine that writes to `ch`. For fan‑out scenarios, let the single producer close its channel after all workers have acknowledged completion, or use a `sync.WaitGroup`.

3. **Use `context` for Cancellation, Not Shared Flags**
   Avoid `bool` flags or `atomic.Bool` to signal cancellation across goroutines. `ctx.Done()` channel integrates with `select` and allows composable, hierarchical cancellation. It also supports deadlines and causes.

4. **Always Drain or Cancel on Early Exit**
   If a goroutine reading from a channel stops early (due to error or `ctx` cancellation), it should either keep draining until channel close (if safe) or arrange for the sender to be cancelled. A common pattern is:
   ```go
   go func() {
       defer func() {
           // Drain channel to avoid sender leak, or cancel ctx
       }()
       for v := range ch { ... }
   }()
   ```
   Better: the sender observes `ctx.Done()` and stops sending.

5. **Prefer Unbuffered Channels for Synchronization**
   For coordination (e.g., signaling completion), an unbuffered channel (`done := make(chan struct{})`) guarantees the signal is received before the sender continues. For data pipelines, start with unbuffered unless benchmarking shows a need for buffering.

6. **Use `sync.WaitGroup` for Fan‑In Coordination**
   When fan‑in merges multiple channels, use a `WaitGroup` to track when all inputs are closed, then close the output channel. This ensures the consumer reads until all producers are finished.

7. **Keep Context Values Minimal and Key‑Safe**
   Only store request‑scoped data (trace IDs, user identity, logger) in context, not optional function parameters. Define unexported key types:
   ```go
   type ctxKey int
   const requestIDKey ctxKey = iota
   ```

8. **Rate Limiter Placement Matters**
   For inbound requests, rate limit at the entry point (e.g., HTTP middleware) before spawning goroutines. For outbound calls to external services, apply a per‑client rate limiter to respect external service limits and avoid cascading failures.

### 8. Examples

#### Complete Pipeline with Cancellation and Error Propagation

```go
package main

import (
    "context"
    "fmt"
    "time"
)

// Stage 1: generates numbers, stops on ctx cancellation
func generate(ctx context.Context, limit int) (<-chan int, <-chan error) {
    out := make(chan int)
    errc := make(chan error, 1)
    go func() {
        defer close(out)
        for i := 0; i < limit; i++ {
            select {
            case out <- i:
            case <-ctx.Done():
                errc <- ctx.Err()
                return
            }
        }
    }()
    return out, errc
}

// Stage 2: squares numbers, may simulate an error
func square(ctx context.Context, in <-chan int) (<-chan int, <-chan error) {
    out := make(chan int)
    errc := make(chan error, 1)
    go func() {
        defer close(out)
        for n := range in {
            if n > 5 { // simulate a condition that fails
                errc <- fmt.Errorf("square: %d too large", n)
                return
            }
            select {
            case out <- n * n:
            case <-ctx.Done():
                errc <- ctx.Err()
                return
            }
        }
    }()
    return out, errc
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
    defer cancel()

    genOut, genErr := generate(ctx, 20)
    sqOut, sqErr := square(ctx, genOut)

    // consume results
    for n := range sqOut {
        fmt.Println(n)
    }
    // check for errors from any stage
    select {
    case err := <-genErr:
        fmt.Println("generator error:", err)
    default:
    }
    select {
    case err := <-sqErr:
        fmt.Println("square error:", err)
    default:
    }
}
```

#### Fan‑In with Error Handling

```go
func merge(ctx context.Context, cs ...<-chan int) (<-chan int, <-chan error) {
    var wg sync.WaitGroup
    out := make(chan int)
    errc := make(chan error, len(cs)) // buffered to avoid blocking

    output := func(c <-chan int) {
        defer wg.Done()
        for n := range c {
            select {
            case out <- n:
            case <-ctx.Done():
                return
            }
        }
    }

    wg.Add(len(cs))
    for _, c := range cs {
        go output(c)
    }

    go func() {
        wg.Wait()
        close(out)
        close(errc)
    }()
    return out, errc
}
```

#### Rate‑Limited HTTP Fetcher

```go
func fetchWithLimit(ctx context.Context, urls []string, reqPerSec int) {
    limiter := rate.NewLimiter(rate.Limit(reqPerSec), reqPerSec) // burst = reqPerSec
    var wg sync.WaitGroup
    for _, u := range urls {
        u := u // capture
        if err := limiter.Wait(ctx); err != nil {
            log.Println("rate limit wait cancelled:", err)
            break
        }
        wg.Add(1)
        go func() {
            defer wg.Done()
            resp, err := http.Get(u) // production: pass ctx via http.NewRequestWithContext
            if err != nil {
                log.Println("fetch error:", err)
                return
            }
            resp.Body.Close()
            log.Println("fetched", u, resp.Status)
        }()
    }
    wg.Wait()
}
```

### 9. Summary & Exercises

This chapter explored how goroutines and channels compose into robust concurrency patterns: pipelines for sequential data processing, fan‑out/fan‑in for workload distribution and aggregation, worker pools for controlled parallelism, rate limiters for protecting external resources, and `context.Context` for carrying deadlines, cancellation, and request‑scoped values. Each pattern reinforces Go’s philosophy of “share memory by communicating,” making concurrent code readable, composable, and less error‑prone.

**Exercises**

1. **Concurrent Web Crawler with Bounded Parallelism and Cancellation**
   Build a web crawler that, given a seed URL, fetches pages and extracts links (using `golang.org/x/net/html`). It must:
   - Limit the number of concurrent fetches to 8.
   - Avoid re‑crawling the same URL (use a map).
   - Respect a per‑domain rate limit of 1 request per second.
   - Accept a `context.Context` for cancellation and gracefully shut down all goroutines on cancel, printing how many URLs were successfully visited.
   - Use a worker pool pattern where workers consume from a URL channel and publish results (links) to a fan‑in channel for deduplication and further enqueueing.

2. **Fan‑Out/Fan‑In with Error Propagation**
   Implement a pipeline that reads lines from a file, hashes each line with SHA‑256 in a fan‑out stage (multiple goroutines), and then writes the hex‑encoded hashes in order. Use a fan‑in stage to collect results. The challenge: preserve original line order. Hint: index each line before fan‑out, and sort by index after fan‑in. Ensure that if any hashing goroutine encounters an error (e.g., simulated empty line), the entire pipeline cancels and returns the error.

3. **Publish/Subscribe System with Slow‑Consumer Protection**
   Design a `PubSub` broker that:
   - Allows subscribers to register and unregister dynamically.
   - Supports topic‑based filtering (subscribe to a string topic).
   - Guarantees that a slow subscriber does not block the publisher; messages to a slow subscriber are dropped after a configurable buffer size, and the broker logs a warning using `slog`.
   - Supports graceful shutdown via context cancellation that unregisters all subscribers and drains pending messages.
   - Write a test that creates 100 subscribers, publishes 1000 messages rapidly, and verifies that no publisher goroutine blocks for more than 10ms.
