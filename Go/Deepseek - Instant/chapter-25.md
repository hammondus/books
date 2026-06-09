# Chapter 25: Advanced Concurrency Patterns

## Basic Usage

Go’s concurrency primitives—goroutines and channels—are simple, but their composition enables powerful patterns. This section demonstrates four foundational patterns: worker pools, pipelines, fan‑in/fan‑out, and rate limiting with backpressure.

### Worker Pool

A worker pool limits concurrent work to a fixed number of goroutines, preventing resource exhaustion.

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

// WorkerPool processes jobs with a fixed number of workers.
func WorkerPool(ctx context.Context, jobs <-chan int, results chan<- int, workers int) {
    var wg sync.WaitGroup
    wg.Add(workers)

    for i := 0; i < workers; i++ {
        go func(workerID int) {
            defer wg.Done()
            for job := range jobs {
                select {
                case <-ctx.Done():
                    return
                default:
                    // Simulate work
                    result := job * 2
                    results <- result
                }
            }
        }(i)
    }

    // Close results channel when all workers finish
    go func() {
        wg.Wait()
        close(results)
    }()
}

func main() {
    jobs := make(chan int, 10)
    results := make(chan int, 10)
    ctx := context.Background()

    // Start worker pool with 3 workers
    go WorkerPool(ctx, jobs, results, 3)

    // Send jobs
    for i := 1; i <= 5; i++ {
        jobs <- i
    }
    close(jobs)

    // Collect results
    for res := range results {
        fmt.Println("Result:", res)
    }
}
```

### Pipeline

A pipeline chains stages, where each stage reads from an input channel and writes to an output channel.

```go
// Stage 1: Generate numbers
func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

// Stage 2: Square numbers
func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

// Stage 3: Sum squares (fan-in)
func sum(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        total := 0
        for n := range in {
            total += n
        }
        out <- total
        close(out)
    }()
    return out
}

func main() {
    // Pipeline: generate → square → sum
    pipeline := sum(square(generate(1, 2, 3, 4)))
    fmt.Println("Sum of squares:", <-pipeline)
}
```

### Fan‑in and Fan‑out

Fan‑out: multiple goroutines read from the same channel. Fan‑in: multiple channels are merged into one.

```go
// fanIn merges multiple channels into a single channel.
func fanIn(ctx context.Context, channels ...<-chan string) <-chan string {
    out := make(chan string)
    var wg sync.WaitGroup

    multiplex := func(ch <-chan string) {
        defer wg.Done()
        for v := range ch {
            select {
            case out <- v:
            case <-ctx.Done():
                return
            }
        }
    }

    wg.Add(len(channels))
    for _, ch := range channels {
        go multiplex(ch)
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

### Rate Limiting with Backpressure

A token bucket pattern implemented with a ticker provides rate limiting.

```go
// RateLimiter allows at most 'rate' operations per second.
type RateLimiter struct {
    ticker *time.Ticker
    sem    chan struct{}
}

func NewRateLimiter(rate int) *RateLimiter {
    interval := time.Second / time.Duration(rate)
    return &RateLimiter{
        ticker: time.NewTicker(interval),
        sem:    make(chan struct{}, 1),
    }
}

func (rl *RateLimiter) Wait(ctx context.Context) error {
    select {
    case <-rl.ticker.C:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

## Under the Hood

### How Channels Enable These Patterns

Channels are **synchronised, typed conduits**. Their blocking nature on send/receive creates natural coordination without explicit mutexes. The compiler‑generated runtime for channels (`runtime/chan.go`) implements a **locking ring buffer** for buffered channels and a **direct handoff** for unbuffered ones.

When you create a pipeline stage:

```go
func square(in <-chan int) <-chan int {
    out := make(chan int)    // unbuffered
    go func() {
        for n := range in {
            out <- n * n      // blocks until receiver ready
        }
        close(out)
    }()
    return out
}
```

The unbuffered channel forces each stage to **wait for the next stage**. This creates implicit backpressure: a slow consumer slows its producer automatically. This is why Go pipelines are naturally flow‑controlled without additional code.

### Goroutine Scheduling in Fan‑out/Fan‑in

When you fan‑out to multiple workers, the Go scheduler distributes these goroutines across OS threads (GOMAXPROCS). Each worker performs a `range` over the shared input channel. The channel itself serialises access: the runtime ensures that each value is delivered to exactly one worker. This is implemented as a **linked list of waiting goroutines** inside the channel’s `recvq` and `sendq`.

Fan‑in with `select` inside a loop is efficient because the `select` statement is compiled into a **linear scan of cases** (O(n) where n is number of channels). For many channels, consider using `reflect.Select` or restructuring.

### Work Stealing in Worker Pools

A naïve worker pool that pre‑assigns jobs (e.g., with a `[]job` slice) suffers from **load imbalance**. Go’s runtime doesn’t automatically rebalance jobs across workers. However, using a **buffered channel as a job queue** combined with `select` and `default` can implement a simple work‑stealing:

```go
func stealWork(jobs chan int, local []int) {
    select {
    case job := <-jobs:
        process(job)
    default:
        // No global jobs, do local work
    }
}
```

True work‑stealing is more complex; the Go scheduler itself uses work‑stealing for goroutines, but not for user‑defined worker pools.

## Why This Design?

### Channels over Shared Memory

The patterns above rely on **communicating sequential processes** (CSP), not shared memory. CSP, formalised by Tony Hoare, treats channels as first‑class citizens. Go’s adoption of CSP prioritises **composition** over **protection**.

Why? Shared memory + locks leads to:

- **Lock contention** and poor scaling.
- **Inversion of control** – the locking logic scatters.
- **Deadlocks** that require global reasoning.

Channels invert this: data flows through explicit pipelines. Ownership becomes clear – a goroutine that writes to a channel may later close it, signalling completion.

### No Built‑in Async/Await

Languages like Rust, JavaScript, and C# offer `async`/`await`. Go deliberately omits this, because goroutines are **already lightweight** and **preemptively scheduled**. An `async` function would be redundant: `go func()` is the primitive. Colourless functions (no distinction between sync/async) simplify APIs.

### Pipeline Design as Function Composition

The pipeline pattern mimics Unix pipes (`|`). Each stage is a pure transformation from `<-chan T` to `<-chan U`. This encourages **testability** (each stage tests in isolation) and **reusability** (stages compose arbitrarily). Go’s decision to make channels **reference types** (like maps and slices) allows efficient passing without copying.

## Competing Approaches

| Language | Pattern | Trade‑off |
|----------|---------|------------|
| **Java** | `ExecutorService`, `CompletableFuture`, `Stream.parallel()` | Heavy abstraction; thread‑based (OS threads); explicit futures require `.join()`. No built‑in backpressure. |
| **Rust** | `tokio` (async tasks), `rayon` (data parallelism), `crossbeam` channels | Zero‑cost abstractions; async/await colours functions; ownership ensures safety, but steeper curve. |
| **Python** | `concurrent.futures`, `asyncio.Queue`, `multiprocessing` | GIL limits threads; asyncio requires `await` everywhere; channels are not native (use `asyncio.Queue`). |
| **C++** | `std::async`, `std::thread`, `boost::asio`, TBB | Manual lifetime management; no garbage collection; powerful but unsafe. |
| **Go** | Goroutines, channels, `select` | Lightweight (～2KB stack), preemptive, channel idioms built‑in. GC overhead may matter for extreme low‑latency. |

**Example comparison: worker pool**

Java (using `ExecutorService`):

```java
ExecutorService pool = Executors.newFixedThreadPool(4);
List<Future<Integer>> futures = new ArrayList<>();
for (int job : jobs) {
    futures.add(pool.submit(() -> job * 2));
}
for (Future<Integer> f : futures) {
    results.add(f.get()); // blocks per future
}
```

Go version (earlier example) is shorter, avoids per‑result blocking, and naturally handles cancellation via context.

## Common Mistakes

### 1. Goroutine Leaks in Pipelines

If a pipeline stage never drains its input channel, the upstream stage blocks forever.

```go
// BAD: Stage that conditionally consumes values
func badStage(in <-chan int) {
    for v := range in {
        if v%2 == 0 {
            fmt.Println(v)
        } else {
            return  // leaks goroutine: channel not drained
        }
    }
}
```

**Fix:** Always drain or propagate cancellation via `context`.

### 2. Deadlock from Unbuffered Channel Dependencies

```go
// DEADLOCK
func main() {
    ch := make(chan int)
    ch <- 42  // blocks forever (no receiver)
    go func() { fmt.Println(<-ch) }()
}
```

**Fix:** Use buffered channel or launch receiver before send.

### 3. Unbounded Worker Queues

Using an unbuffered channel for jobs in a worker pool is safe, but a **very large buffered channel** can hide memory exhaustion. If producers outrun consumers, the buffer grows until OOM.

```go
// POTENTIAL MEMORY BOMB
jobs := make(chan int, 1_000_000) // static large buffer
```

**Fix:** Either use unbuffered (natural backpressure) or bounded with drop policy using `select` with `default`:

```go
select {
case jobs <- task:
default:
    // Drop task or apply backpressure to producer
}
```

### 4. Closing Channels Prematurely

Closing a channel that still has senders causes panic. Closing a channel from a receiver side breaks the `range` contract.

**Idiom:** The **writer** closes a channel; the **reader** never does. For fan‑in, use a `sync.Once` or `sync.WaitGroup` to coordinate closing.

## Performance Considerations

### Channel Throughput

A channel send/receive pair costs roughly **50‑100 ns** on modern hardware (single producer/single consumer). Contention from multiple goroutines increases cost due to atomic operations and queuing.

| Operation | Approximate cost (ns) |
|-----------|----------------------|
| Unbuffered chan send (no wait) | ~50 |
| Buffered chan send (buffer available) | ~40 |
| Select on 2 channels | ~100 |
| Mutex Lock/Unlock (uncontended) | ~30 |

### Fan‑out Overhead

Fan‑out to N workers adds **N goroutine stacks** (each ~2KB) and scheduler overhead. For high throughput (millions of jobs), consider:

- **Batching:** Send slices of jobs instead of individual items.
- **Fixed worker count** bound to `runtime.GOMAXPROCS(0)`.
- **Avoid `select`** in tight loops if only one channel is active.

### Memory Allocations

Channels themselves allocate on the heap (the `hchan` struct). Each send of a value **copies** the value into the channel’s buffer (or directly to receiver’s stack). For large structs, this copy can dominate cost. Prefer sending pointers or indices when appropriate.

### Backpressure and Saturation

A pipeline with a slow stage causes goroutines to pile up on channel sends. Each blocked goroutine consumes stack memory. This is **intentional** (backpressure), but unbounded blocking can lead to **goroutine leaks** if the pipeline never unblocks. Use `select` with `time.After` or `context` to fail fast.

## Best Practices

### 1. Own the Channel Lifetime

Define clearly which goroutine **closes** a channel. Document it.

```go
// Produce sends values on out and then closes it.
func Produce(out chan<- int) { ... }
```

### 2. Use `context` for Cancellation Propagate

All long‑running goroutines in a pipeline should respect `ctx.Done()`.

```go
func stage(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range in {
            select {
            case out <- transform(v):
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}
```

### 3. Limit Concurrency with Worker Pools, Not `sync.WaitGroup`

For unbounded goroutine creation (e.g., handling HTTP requests), always bound the number via a worker pool or a semaphore channel.

```go
// Semaphore pattern to limit concurrency
var sem = make(chan struct{}, 10)

func handleRequest() {
    sem <- struct{}{}        // acquire
    defer func() { <-sem }() // release
    // do work
}
```

### 4. Prefer Unbuffered Channels by Default

Unbuffered channels give **synchronous handoff** and clear backpressure. Add a buffer only after profiling shows it reduces contention without causing queue bloat.

### 5. Name Channels After Their Content

`jobs <-chan int`, `results <-chan *Result`. Avoid generic `ch`.

### 6. Avoid Global Channels

Global channels create implicit coupling. Pass channels as parameters to functions, just like any other dependency.

## Examples

### Example 1: Parallel File Checksum (Fan‑out, Fan‑in)

```go
func checksumFiles(ctx context.Context, files []string) (map[string]uint64, error) {
    jobs := make(chan string)
    results := make(chan struct{
        path string
        sum  uint64
        err  error
    })

    // Worker pool
    const workers = 4
    var wg sync.WaitGroup
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for path := range jobs {
                sum, err := computeChecksum(ctx, path)
                results <- struct{ ... }{path, sum, err}
            }
        }()
    }

    // Send jobs
    go func() {
        for _, f := range files {
            select {
            case jobs <- f:
            case <-ctx.Done():
                close(jobs)
                return
            }
        }
        close(jobs)
    }()

    // Close results after workers finish
    go func() {
        wg.Wait()
        close(results)
    }()

    // Collect results
    out := make(map[string]uint64)
    for r := range results {
        if r.err != nil {
            return nil, r.err
        }
        out[r.path] = r.sum
    }
    return out, nil
}
```

### Example 2: Rate‑Limited API Scraper with Backpressure

```go
type RateLimiter struct {
    ticker *time.Ticker
    mu     sync.Mutex
    last   time.Time
}

func (rl *RateLimiter) Wait(ctx context.Context) error {
    rl.mu.Lock()
    defer rl.mu.Unlock()
    select {
    case <-rl.ticker.C:
        rl.last = time.Now()
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

func ScrapeWithRateLimit(ctx context.Context, urls []string, rate int) <-chan *Result {
    out := make(chan *Result, len(urls))
    rl := NewRateLimiter(rate)
    go func() {
        defer close(out)
        var wg sync.WaitGroup
        for _, url := range urls {
            if err := rl.Wait(ctx); err != nil {
                return
            }
            wg.Add(1)
            go func(u string) {
                defer wg.Done()
                res, err := fetch(ctx, u)
                out <- &Result{URL: u, Data: res, Err: err}
            }(url)
        }
        wg.Wait()
    }()
    return out
}
```

## Summary & Exercises

### Summary

- **Worker pools** bound concurrency and protect against resource saturation.
- **Pipelines** compose sequential transformations using channels, providing natural backpressure.
- **Fan‑out/Fan‑in** distribute and collect work, ideal for parallel processing.
- **Rate limiting** and **backpressure** are built via tickers and buffered channels.
- **Context** is the standard mechanism for cancellation and deadlines across all patterns.
- Common mistakes include goroutine leaks, deadlocks, and unbounded queues.

### Exercises

**Exercise 1: Build a concurrent prime sieve.**  
Implement a pipeline that generates integers from 2 to N, filters out non‑primes using the Sieve of Eratosthenes, and prints primes. Each filter stage should be a separate goroutine. Ensure cancellation stops the entire pipeline when N is reached.

**Exercise 2: Implement a publish‑subscribe (pub/sub) system.**  
Write a `Broker` type with `Subscribe(topic string) <-chan Message` and `Publish(topic string, msg Message)` methods. Topics should be dynamic. Use fan‑in to deliver messages to all subscribers of a topic. Handle subscriber slow consumption by dropping messages (provide a `WithDropPolicy` option). Measure the performance impact of a `map[string][]chan Message`.

**Exercise 3: Rate‑limited worker pool with load shedding.**  
Create a `JobScheduler` that accepts jobs (functions) and processes them with a fixed concurrency limit and a global rate limit (e.g., 100 jobs/second). If the queue exceeds a configurable length (backpressure), the scheduler should reject new jobs with an error (load shedding). Use only channels and `context`. Compare throughput with a naive unbounded queue using `go test -bench`.

---

*In the next chapter, we’ll explore synchronisation primitives (`Mutex`, `WaitGroup`, atomic operations) and understand when channels are the wrong tool – and how to detect race conditions.*
