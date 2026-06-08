## Chapter 23. Channels & Communication

Channels are the linguistic embodiment of Go’s signature concurrency mantra: “Share memory by communicating.” While the previous chapter established the runtime scheduler and goroutine mechanics, channels are the constructs that turn a pile of independent goroutines into a coordinated, composable system. They provide a typed, synchronized conduit through which goroutines can pass data—or simply signal events—without reaching for explicit locks. Done well, channel design keeps data ownership clear, eliminates entire classes of race conditions, and makes concurrent code both readable and maintainable. This chapter dissects everything from the most basic send/receive syntax down to the runtime internals, so you can wield channels with precision rather than intuition.

---

### 1. Basic Usage

A channel is a first-class value created with `make`, parametrized by the type of value it transports.

```go
// Unbuffered channel: send blocks until a receive is ready, and vice versa.
ch := make(chan int)

// Buffered channel with capacity 3; sends block only when the buffer is full.
ch := make(chan string, 3)
```

Sending and receiving use the `<-` operator, which visually suggests data flow. A send places the arrow to the right of the channel; a receive places it to the left.

```go
ch <- 42          // send
v := <-ch         // receive
v, ok := <-ch     // receive with “comma ok” – ok is false if channel is closed and empty
```

The built-in `close` shuts down a channel, causing subsequent receives to return the zero value immediately with `ok == false`.

```go
close(ch)
```

A `for range` loop over a channel automatically drains it until it is closed, making it the canonical consumption pattern:

```go
for msg := range ch {
    fmt.Println(msg)
}
// Loop exits when ch is closed and all buffered elements have been received.
```

Channels can be **directional**, restricting the operations available in a function signature. This is not just documentation—the compiler enforces it.

```go
func producer(out chan<- int) { // send-only
    for i := 0; i < 5; i++ {
        out <- i
    }
    close(out)
}

func consumer(in <-chan int) { // receive-only
    for v := range in {
        fmt.Println(v)
    }
}
```

Even a bidirectional channel can be implicitly converted to a directional one when passed as an argument. The reverse is impossible, which protects invariants.

A simple synchronous handoff between two goroutines illustrates the “meeting point” nature of an unbuffered channel:

```go
func main() {
    ch := make(chan string)
    go func() {
        ch <- "hello"     // blocks until main is ready to receive
    }()
    msg := <-ch           // blocks until the goroutine sends
    fmt.Println(msg)
}
```

The program always prints “hello” and exits cleanly because the send and receive synchronize—no `sync.WaitGroup` needed for this one-shot coordination.

---

### 2. Under the Hood

Each channel is represented at runtime by an `hchan` struct (defined in `runtime/chan.go`). Understanding its layout demystifies performance and behaviour.

- **`buf`**: a pointer to a circular queue that holds buffered elements. The buffer is a contiguous array allocated alongside the `hchan` if the channel is buffered.
- **`sendx` / `recvx`**: indices into the buffer ring for the next send and next receive.
- **`lock`**: a lightweight mutex (`mutex`) that protects the channel’s internal state. Every send, receive, or close acquires this lock.
- **`recvq` / `sendq`**: wait queues of goroutines (parked in `sudog` structs) that are blocked waiting to receive or send, respectively.
- **`closed`**: a flag indicating the channel has been closed.

When you send on an **unbuffered** channel, the runtime briefly acquires the lock. If a receiver is already waiting in `recvq`, the sender copies the value directly into the receiver’s stack (no intermediate buffer), wakes the receiver, and releases the lock. This direct handoff is exceptionally cheap. If no receiver is waiting, the sender parks itself on `sendq`, giving up its P and M, until a receiver arrives. The symmetry holds for receives.

A **buffered** channel behaves like a bounded lock-free-ish queue—though the internal operations are lock-protected, they are short. If the buffer has room, the send copies the value into `buf[sendx]`, advances the index, and completes. Only when the buffer is full does the sender park. A receive on a non-empty buffer copies the value out, advances `recvx`, and then checks `sendq`: if a blocked sender is waiting, the runtime copies the sender’s value directly into the buffer (bypassing the normal enqueue) and wakes the sender. This optimisation avoids an extra copy and context switch.

**Closing** sets the `closed` flag, then wakes every goroutine in `recvq`. Those receivers will complete with the zero value and `ok == false`. Any goroutine blocked in `sendq` is woken as well but will immediately panic because sending on a closed channel is illegal. The runtime does *not* drain the buffer on close; buffered elements can still be received normally until exhausted, after which receives yield the zero value.

Channel operations are **not** atomic in the lock‑free sense, but the combination of a fast user‑space mutex, short critical sections, and careful scheduling integration means that uncontended sends/receives incur only a few tens of nanoseconds of overhead. The real cost appears under contention, where goroutine parking and unparking involve system calls (e.g., `futex` on Linux).

Crucially, channel semantics integrate with the Go memory model: a **send happens-before the corresponding receive** that completes that send. This ensures that all memory writes performed before the send are visible to the goroutine that performs the receive. Conversely, a **close happens-before a receive that returns a zero value because the channel is closed**. These guarantees make channels a proper synchronisation primitive without any additional atomics.

---

### 3. Why This Design?

Channels exist because the Go team observed that shared memory with mutexes, while powerful, scales poorly in terms of developer comprehension. When a dozen goroutines access the same map with a lock, reasoning about which goroutine “owns” the data, when the lock must be held, and what invariants hold between lock and unlock becomes a mental burden. Channels invert this: you **pass data** to a goroutine that then owns it exclusively, or you **signal** an event. The data flows; ownership is transferred.

Several deliberate design choices distinguish Go:

- **Language-integrated syntax** rather than a library. `<-` and `select` (next chapter) are part of the grammar, not just function calls. This allows the compiler to perform escape analysis specially for channel operations, and enables the static analysis tools (`go vet`) to catch mistakes like sending on a receive-only channel. It also makes channel use as natural as assignment.
- **Explicit `close`** rather than reference counting or automatic destruction. The sender is expected to close the channel when it is done producing, broadcasting a “done” signal to all receivers. This explicit ownership model eliminates the question “who is still using this?” that plagues actor-mailbox shutdown in other languages. The price is discipline—closing from the wrong goroutine panics.
- **Typed channels** that carry a single element type. This avoids the boxing and type-assertion churn of generic message queues found in many actor frameworks. When you need polymorphic messages, you can send an interface value (e.g., `any`), but the common case—pipeline of bytes, a stream of requests—stays fast and clear.
- **Buffered channels** are part of the core, not an afterthought. They let you decouple sender and receiver, absorbing bursts. Yet the capacity is always explicit in the `make` call, forcing the programmer to decide on a bound. This prevents the unbounded queue problem that can crash systems under load.
- **No built-in notion of process or actor identity**. Channels are values that can be stored in maps, passed over other channels, or shared across goroutines freely. This gives enormous flexibility—you can build fan‑in/fan‑out topologies, dynamic pipelines, or publish/subscribe systems without any runtime support beyond `chan`. The trade‑off is that the programmer must enforce higher-level invariants (e.g., not closing a channel that is still being sent to by another goroutine).

Compare this with the alternative of providing only mutexes and condition variables (as in pthreads): you would have to build your own rendezvous, buffering, and broadcast semantics, with all the pitfalls of manual memory ordering. Channels are a carefully designed abstraction that raises the level of discourse from “lock/unlock” to “send/receive”.

---

### 4. Competing Approaches

To appreciate Go’s model, it helps to see how other ecosystems solve the same problem.

- **Python** offers `queue.Queue` for thread-based concurrency and `asyncio.Queue` for coroutines. Both are library constructs, not language primitives. The GIL limits true parallelism, so a Python queue is often used to shuttle work between threads that are not CPU-bound, making the synchronisation cost less of a concern. However, the lack of a `select`-like construct forces you to poll with timeouts or use multiple threads. Go’s channels are simpler, faster, and integrated with a scheduler that can take advantage of all CPU cores.
- **Java**’s `java.util.concurrent` provides `BlockingQueue` implementations (like `LinkedBlockingQueue`, `SynchronousQueue`). They are feature‑rich—fairness policies, capacity, drain‑to—but their API is object‑oriented, requiring method calls (`put`, `take`). Java does not have a direct analogue of Go’s `select`; you must use `ExecutorService` and `CompletionService` or write bespoke polling loops. The result is more ceremony. Go’s minimalism pushes you toward a uniform pattern: channels + `select`.
- **C++** has no standard channel. Building one from `std::queue`, `std::mutex`, and `std::condition_variable` is a common exercise. Boost.Lockfree provides `spsc_queue` for lock‑free single‑producer/single‑consumer scenarios, but it sacrifices blocking semantics. C++20 coroutines and `libunifex` can model async streams, but the ecosystem is fragmented. Go offers a single, clean, compiler‑blessed abstraction that works both for synchronous handoffs and buffered pipelines.
- **Rust**’s `std::sync::mpsc` and the crossbeam crate provide multi‑producer, single‑consumer channels with strong ownership semantics. Sending a value moves it; the receiver is the sole owner afterwards. This eliminates the risk of data races on the sent data itself. Go, by contrast, copies the value (or copies the pointer). If you send a pointer or a struct containing a map, concurrent access remains dangerous. Rust’s compile‑time checks guarantee no races; Go trusts the programmer to follow the rule “after sending, don’t touch the data unless you’re sure.” The trade‑off is productivity versus static safety. Both are valid, but they reflect fundamentally different philosophies about how much the language should enforce.
- **JavaScript** lacks shared‑memory threads for the most part. Web Workers communicate via `postMessage`, which serializes and copies data. Channels are not applicable in the same sense, though Node.js `worker_threads` with `MessageChannel` approach the pattern. Go’s goroutines share a single address space, so channels transmit pointers without copying large payloads—a performance characteristic that serialization‑based systems cannot match.

In summary, Go’s channels win on integration, simplicity, and performance for in‑process messaging. They lose out on compile‑time race freedom (compared to Rust) and on the rich policy knobs of Java’s queues. The Go way is to start with channels and only fall back to lower‑level primitives when profiling demands it.

---

### 5. Common Mistakes

Channels are simple but unforgiving. The following pitfalls catch nearly every seasoned engineer new to Go.

**Sending on a closed channel** – This panics immediately. It is the most frequent surprise because the sender may not be the goroutine that called `close`. The solution is strict **channel ownership**: the goroutine that creates a channel should also be the one that closes it, and it must ensure no further sends occur.

**Closing a channel from the receiver side** – If the receiver closes the channel and the sender subsequently tries to send, the sender panics. This is a design error; receivers should only signal completion by returning or using a separate “done” channel.

**Forgetting to close a channel** – When a goroutine uses `for range` on a channel, the loop blocks indefinitely unless the channel is closed. The goroutine will never exit, leaking both the goroutine and any memory it references. A common variant: a pipeline where the upstream producer exits without closing, and the downstream consumer hangs.

**Deadlock on unbuffered sends** – An unbuffered send demands a simultaneous receive. If no goroutine is ready, the program deadlocks. This often happens when a main goroutine sends before launching the corresponding receiver goroutine.

**Sharing mutable data after sending** – Sending a struct containing a slice, map, or pointer does not transfer ownership of the underlying data; it transfers a copy of the header. If the sender continues to mutate the backing array or map, a data race ensues. To safely transfer ownership, send a deep copy, or design your types to be immutable after sending.

**Using buffered channels as unbounded queues** – A buffered channel has a fixed capacity, chosen at creation. If producers permanently outpace consumers, the buffer fills up and producers block. This is often desirable (backpressure), but if the buffer capacity is set to an arbitrarily large number to avoid blocking, the channel becomes a memory time bomb. Use a proper queue (e.g., a linked list protected by a mutex) if you truly need unbounded buffering.

**Nil channel deadlocks** – Both sending and receiving on a nil channel block forever. This behaviour is sometimes useful in `select` to disable a case, but a forgotten initialisation leads to a silent goroutine hang.

**`select` with `default` in a tight loop** – Using `default` to make a channel operation non‑blocking is fine for polling, but if the loop has no back‑off, it burns a CPU core. This mistake surfaces frequently when converting asynchronous code from languages where non‑blocking I/O is the norm.

**Assuming channel operations are lock‑free** – They are not. Under high contention, channel operations can become a bottleneck because the internal lock serialises access. Never use a single channel as a high‑throughput work queue for hundreds of CPU‑bound goroutines; fan‑out or shard the channel in such cases.

---

### 6. Performance Considerations

Understanding the overhead of channels lets you decide when they are the right tool and when to reach for atomics or mutexes.

**Allocation** – A `make(chan T, N)` allocates an `hchan` struct plus a contiguous buffer of size `N * sizeof(T)`. For unbuffered channels, no buffer is allocated. The struct itself is small (a few words) and will often be stack‑allocated if the channel does not escape. Passing pointers through channels causes those values to escape to the heap, increasing GC pressure. Prefer sending value types (small structs) when possible to keep allocations low.

**Copy cost** – Every send/receive copies the element. For large structs, this cost can be significant. Sending a pointer avoids the large copy but introduces sharing and allocation. In many cases, a small, immutable struct is the best trade‑off.

**Lock contention** – The channel’s internal mutex means that a single channel scales poorly when many goroutines hammer it concurrently. Each operation acquires the lock, does a short critical section, and releases it. With enough concurrency, this becomes a point of contention. Sharding (multiple channels plus a deterministic routing key) can restore scalability.

**Goroutine parking** – When a goroutine blocks on a channel, it is parked (descheduled) and then unparked when the operation can proceed. Parking and unparking involve system calls (e.g., `futex`) and scheduler work, costing several microseconds in the worst case. This is much heavier than a simple atomic compare‑and‑swap. For ultra‑low‑latency signaling, consider `sync/atomic` or a `sync.WaitGroup` for one‑shot events.

**Unbuffered vs. buffered** – Unbuffered channels perform a direct value copy from sender stack to receiver stack (or vice versa) under the hood, avoiding an intermediate buffer. This is often faster than buffered channels when the goroutines are already paired. Buffered channels add buffer management and an extra copy (into the buffer and then out of it), but they decouple execution and can improve throughput by reducing synchronisation stalls.

**Select overhead** – A `select` with many cases must lock all the channels involved (in a consistent order to prevent deadlocks). The runtime performs a pseudo‑random shuffle to ensure fairness, which adds CPU cycles. Do not put dozens of cases in a hot `select`; consider a slice of channels and reflection only if necessary, or better, redesign the topology.

**Leaks** – A channel that is never closed, and on which goroutines are blocked, prevents both the channel and the goroutines from being garbage collected. The memory footprint grows with the number of stalled goroutines. Always ensure there is a path that closes the channel or that goroutines can exit via a `context` cancellation or a `done` channel.

**Benchmarking hint** – When measuring channel performance, be aware that the Go runtime employs several optimisations: a send on an unbuffered channel that finds a waiting receiver can bypass the scheduler’s runqueue and hand off the P directly. Micro‑benchmarks should therefore model realistic contention, not just a single pair.

---

### 7. Best Practices

Idiomatic Go channels are not about clever tricks; they are about clarity and safety.

**Establish channel ownership** – The goroutine that creates a channel is responsible for closing it. Communicate this contract through function boundaries: a function that returns a `<-chan T` signals that it will close the channel when it’s done producing. Receivers should treat the channel as read‑only and never close it.

**Use `chan struct{}` for signalling** – When you need to notify an event without sending data, a channel of empty structs allocates no value memory and makes the intent obvious: `done := make(chan struct{})`. Closing it broadcasts the signal to all receivers.

**Close to broadcast** – Because a closed channel returns immediately to all pending and future receives, closing a channel is the canonical way to announce “no more data” or “shutdown.” This pattern powers graceful termination of pipelines.

**Prefer `for range` for consumption** – It clearly expresses “process everything until the source is exhausted” and automatically handles the `ok` boolean.

**Use directional channels in signatures** – A function that accepts `chan<- Msg` documents that it only produces; one that takes `<-chan Msg` documents that it only consumes. The compiler enforces the contract, and the code becomes self‑documenting.

**Buffer consciously** – An unbuffered channel enforces synchronous rendezvous, which is the simplest and safest default. Introduce a buffer only when you need to decouple timing (e.g., to absorb bursts or to let a producer move on before a consumer is ready). Choose a capacity based on your expected burst size and memory budget, not an arbitrary “big number.” A capacity of 1 is often enough for a semaphore or a token.

**Avoid closing channels from the receiver side** – Instead, let the receiver signal “I’m done” by returning or by sending on a separate `done` channel (which the producer may select on). If you must cancel a producer, use `context.Context` and let the producer’s send loop respect `ctx.Done()`.

**Never expose channels directly in APIs unless necessary** – A function that starts a background goroutine and returns a `<-chan Result` is a common and acceptable pattern. But an exported `chan Request` that the caller is expected to write to is fragile—who closes it? How many producers are allowed? Wrap such channels in methods that encapsulate the ownership contract.

**Use `select` with a `default` case sparingly** – It’s appropriate for a non‑blocking send in a telemetry path where dropping a sample is acceptable. It is not appropriate for core business logic without a back‑off.

**Nil channels are a valid design element** – Setting a channel to nil inside a `select` case disables that case for the next iteration, which is useful for implementing state machines. Ensure the channel is properly initialised before the loop starts.

**When sharing mutable state is unavoidable, use a mutex inside a single goroutine** – This pattern—a goroutine owns the state, and all access happens through a channel that sends commands—is a common Go idiom. It keeps the mutex invisible to the rest of the program.

---

### 8. Examples

Let’s move from snippets to a complete, idiomatic example that demonstrates ownership, buffering, and graceful shutdown.

**Example: A typed pipeline with backpressure**

Imagine a pipeline that reads integers from a source, squares them, and prints the result. We’ll use two goroutines connected by a channel. The source owns the channel it sends on and closes it when done. The consumer uses `for range`.

```go
package main

import "fmt"

// source generates numbers and sends them on the returned channel.
// It closes the channel when the generator loop finishes.
func source(n int) <-chan int {
	out := make(chan int) // unbuffered: synchronous handoff
	go func() {
		for i := 0; i < n; i++ {
			out <- i
		}
		close(out)
	}()
	return out
}

// square reads from in, squares each value, and sends to a new channel.
func square(in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		for v := range in {
			out <- v * v
		}
		close(out)
	}()
	return out
}

func main() {
	// Compose: source -> square -> consumer.
	nums := source(5)
	squares := square(nums)

	// Consumer: drain until closed.
	for res := range squares {
		fmt.Println(res)
	}
}
```

Output:
```
0
1
4
9
16
```

Everything exits cleanly because `source` closes its channel, causing `square`’s `for range` to exit, which then closes its own channel, allowing `main` to finish.

**Example: Graceful shutdown with a `done` channel**

A real service often needs to stop producers early. A dedicated `done` channel, coupled with `select`, lets the producer abort.

```go
func cancellableSource(ctx context.Context) <-chan string {
	out := make(chan string)
	go func() {
		defer close(out)
		for _, item := range []string{"a", "b", "c", "d"} {
			select {
			case out <- item:
			case <-ctx.Done():
				return // caller cancelled; stop sending
			}
		}
	}()
	return out
}
```

Here, the producer respects cancellation, and the consumer can rely on the channel eventually being closed (either after all items or early). No goroutine leaks.

**Example: Broadcasting shutdown with `close`**

A single `close` on a channel wakes multiple goroutines. This pattern is useful for tearing down a set of workers.

```go
func worker(id int, shutdown <-chan struct{}, wg *sync.WaitGroup) {
	defer wg.Done()
	for {
		select {
		case <-shutdown:
			fmt.Printf("worker %d shutting down\n", id)
			return
		default:
			// do work
		}
	}
}

func main() {
	shutdown := make(chan struct{})
	var wg sync.WaitGroup
	for i := 0; i < 3; i++ {
		wg.Add(1)
		go worker(i, shutdown, &wg)
	}
	// … later
	close(shutdown) // signal all workers to stop
	wg.Wait()
}
```

No data is sent; `close(shutdown)` acts as a one‑shot broadcast that every goroutine notices instantly.

---

### 9. Summary & Exercises

**Recap**

- Channels are typed, thread‑safe conduits that synchronise goroutines without explicit locks. Unbuffered channels enforce a synchronous rendezvous; buffered channels provide limited decoupling.
- Internally, an `hchan` uses a mutex, a circular buffer, and wait queues of parked goroutines. Direct handoff for unbuffered channels avoids an intermediate copy.
- The design reflects the philosophy “share memory by communicating”: pass data, transfer ownership, avoid shared mutable state. Language integration provides safety and readability that libraries cannot match.
- Compared to Python’s queues, Java’s `BlockingQueue`, or C++’s DIY constructs, Go’s channels are simpler, more uniform, and tightly coupled to the scheduler.
- Common pitfalls include closing from the wrong goroutine, leaking goroutines by forgetting to close, data races on shared references, and misusing buffered channels as unbounded storage.
- Performance is generally excellent, but channel operations involve locking and potential parking. For high‑contention hotspots or simple signalling, atomics or `WaitGroup` may be cheaper.
- Best practices: define ownership, prefer `chan struct{}` for signals, close to broadcast, use directional types, and never close a channel from the receiver side.

**Exercises**

1. **Worker pool with graceful shutdown**
   Build a system that accepts jobs on a `chan Job` and processes them with a fixed pool of N worker goroutines. The pool should support a `Shutdown()` method that causes all workers to finish their current job and then exit cleanly. No goroutines should leak, and no jobs should be lost. How do you handle jobs still in the channel at shutdown? What ownership rules do you establish for the job channel and the shutdown signal?

2. **Bounded parallel executor using a semaphore channel**
   Create a function `func BoundedExec(ctx context.Context, tasks []func() error, concurrency int) []error` that runs all tasks, but limits the number of concurrently executing goroutines to `concurrency`. Use a buffered channel as a semaphore (e.g., `make(chan struct{}, concurrency)`) to restrict concurrency. Ensure the function respects context cancellation, and collect all errors without races. Consider how buffering the result channel might affect backpressure.

3. **Non‑blocking message router with drop policy**
   Design a publish/subscribe hub where multiple producers send messages to a central broker goroutine via a buffered channel, and multiple consumers each have their own buffered channel. If a consumer’s buffer is full, the broker should drop the message for that consumer rather than block. The broker must handle dynamic subscription/unsubscription without data races. Specify the channel directions, ownership, and the mechanism by which the broker detects “full” without blocking (hint: `select` with `default`). Discuss how you would monitor dropped messages for observability.
