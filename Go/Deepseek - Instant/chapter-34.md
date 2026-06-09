## Chapter 34: Debugging Production Systems

Production debugging is where theory meets reality. Your service is live, users are reporting errors, CPU is spiking, memory is climbing, and you have seconds to understand what’s wrong before the pager wakes everyone up. Go’s runtime and tooling provide a rich set of introspection capabilities—**pprof**, **trace**, **core dumps**, and **execution traces**—that allow you to diagnose issues in running systems without restarting or redeploying. This chapter covers how to use those tools effectively, with a focus on low-overhead, production‑safe techniques.

---

### 1. Basic Usage

The Go toolchain includes profiling and debugging facilities that integrate directly into your binary. The most common entry point is `net/http/pprof`, which registers handlers on the default HTTP server.

**Enabling the pprof HTTP endpoint**

```go
package main

import (
    "log"
    "net/http"
    _ "net/http/pprof" // registers handlers on default mux
    "runtime"
    "time"
)

func main() {
    // Optional: enable mutex and block profiling
    runtime.SetMutexProfileFraction(5)   // 1/5 of mutex events
    runtime.SetBlockProfileRate(1000)    // 1/1000 of blocking events

    // Start your application server
    go startAppServer()

    // Expose pprof on a separate port (security best practice)
    go func() {
        log.Println("pprof listening on :6060")
        log.Fatal(http.ListenAndServe(":6060", nil))
    }()

    select {} // block forever
}

func startAppServer() {
    // your actual service logic
}
```

**Capturing a core dump**

On Linux, you can generate a core dump without stopping the process using `gcore`:

```bash
# Get the PID of your Go process
$ pgrep my-service
12345

# Generate core dump
$ gcore 12345
# writes core.12345

# Analyze with delve
$ dlv core ./my-service core.12345
(dlv) bt
(dlv) goroutines
```

**Using `go tool pprof` on a live endpoint**

```bash
# Sample CPU for 30 seconds
$ go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Sample heap (inuse space)
$ go tool pprof -inuse_space http://localhost:6060/debug/pprof/heap

# Sample heap (allocated objects over time)
$ go tool pprof -alloc_objects http://localhost:6060/debug/pprof/heap

# View mutex contention
$ go tool pprof http://localhost:6060/debug/pprof/mutex

# Interactive web UI
$ go tool pprof -http=:8080 http://localhost:6060/debug/pprof/profile?seconds=30
```

**Capturing an execution trace**

Traces show goroutine scheduling, GC events, and blocking operations with microsecond precision.

```go
// In your code, conditionally start tracing
f, _ := os.Create("trace.out")
defer f.Close()
if err := trace.Start(f); err != nil {
    log.Fatal(err)
}
defer trace.Stop()
```

Then analyze with:
```bash
$ go tool trace trace.out
# Opens a web UI with timeline, goroutine analysis, and more
```

---

### 2. Under the Hood

Go’s profiling infrastructure is built into the runtime and requires no external instrumentation. Here’s how it works at the engine level.

**CPU profiling**  
The runtime installs a **SIGPROF** handler that interrupts the process at a fixed frequency (default 100 Hz). On each interrupt, the runtime records the current `PC` (program counter) and stack trace. Sampling is **statistically valid** for hot paths because the interrupt is a Poisson process. The profile is stored as a map from `(stack, PC)` to sample count.

**Heap profiling**  
The memory allocator (`malloc.go`) tracks live objects. When you request a heap profile, the runtime walks the heap, records stack traces for each allocation (up to a configurable depth), and aggregates by size and count. The default memory profile sample rate is 1 allocation per 512 KB – high enough to be useful without killing performance.

**Mutex and block profiling**  
These use a different mechanism: when a goroutine blocks on a mutex or channel operation, the runtime records the time spent waiting. The `SetMutexProfileFraction` and `SetBlockProfileRate` control what fraction of events are sampled. Setting these to 1 captures everything (dangerous in production).

**Core dumps and `GOTRACEBACK`**  
When a Go process crashes (panic, segfault, or `SIGABRT`), the runtime dumps a stack trace for all goroutines. The `GOTRACEBACK` environment variable controls verbosity:

- `GOTRACEBACK=none` – single goroutine stack
- `GOTRACEBACK=single` – default, current goroutine only
- `GOTRACEBACK=all` – all goroutines
- `GOTRACEBACK=system` – all goroutines + runtime frames

For core dumps, Go writes a standard ELF core file containing the full memory image. However, Go’s garbage collector compacts the heap, so traditional gdb may show stale pointers. Use **delve** (`dlv core`) instead, which understands Go’s memory layout and goroutine stacks.

**Execution traces**  
The tracer uses **event logging** – every goroutine creation, channel send/receive, system call, and GC event writes to a circular buffer. The overhead is higher than pprof (≈10–20% CPU), but the data is deterministic and allows replay. The trace format is a stream of timestamped events; the `go tool trace` viewer reconstructs the scheduler’s behavior.

---

### 3. Why This Design?

Go’s debugging story reflects three core philosophies: **self‑containment**, **operational simplicity**, and **production safety**.

**Self‑contained profiling**  
Unlike Java’s JVMTI agents (which require loading separate `.so` files and often force restarts) or Python’s `py-spy` (which relies on `ptrace`), Go’s profiler lives inside the runtime. You don’t deploy extra binaries, open special ports, or modify startup scripts. The profiling handlers are just HTTP handlers – the same infrastructure that serves your API can serve profiling data.

**No restarts required**  
Because profiling is always on at a low level (the sampling ticks are always running), you can activate high‑detail profiles (e.g., mutex profiling) at runtime via `runtime.SetMutexProfileFraction`. This is critical for debugging transient issues that disappear after a restart.

**Trade‑off: Overhead vs. Fidelity**  
The Go team chose **statistical sampling** over deterministic tracing for most profiles. This keeps overhead low (1–3% for CPU profiles, <1% for heap profiles) but means you cannot guarantee that every allocation or every blocking event is captured. For high‑stakes debugging, you can dial up the sample rate, but the default strikes a pragmatic balance.

**Core dumps as a last resort**  
Go deliberately does not encourage core dumps for routine debugging. The heap is compacted, pointers are rewritten, and stacks are moved – a core dump is a snapshot of a moving target. Instead, the runtime encourages **live introspection** via pprof. Core dumps exist for crashes that are impossible to reproduce live (e.g., kernel bugs, memory corruption in cgo).

---

### 4. Competing Approaches

| Language / Platform | Primary Tools | Overhead | Live Debugging | Remote Capability |
|---------------------|---------------|----------|----------------|--------------------|
| **Go** | pprof (HTTP), trace, delve | 1–5% (CPU) | Yes (no restart) | HTTP + `go tool pprof` |
| **Java** | JProfiler, async‑profiler, JMC | 2–10% (depending on agent) | Yes, but often requires JMX or attaching agent | JMX over RMI, or flight recorder |
| **Python** | py‑spy, cProfile, `sys.setprofile` | 10–30% (cProfile); py‑spy lower | Limited – many tools need restart or `ptrace` | Remote via debugpy (slow) |
| **Node.js** | inspector, 0x, clinic | 10–40% (inspector) | Yes (inspector protocol) | Chrome DevTools remote |
| **Rust** | perf, `pprof-rs`, tokio‑console | <2% (perf sampling) | Limited – perf requires root; tokio‑console is opt‑in | No standard remote profile endpoint |

**Key differences**

- **Java** prioritizes deep introspection (method‑level profiling, object allocation traces) but at the cost of complexity. Attaching a Java agent may require restarting the JVM or using `-XX:+FlightRecorder`, which is powerful but heavy.
- **Python**’s `cProfile` is deterministic and thus high‑overhead; `py-spy` is sampling but relies on `ptrace`, which can be blocked by container security policies. Live debugging often requires `pdb` in a terminal, which stops the process.
- **Rust** defers profiling to system tools (`perf`) or library‑level instrumentation. There is no built‑in HTTP endpoint; you must wire up `pprof-rs` manually. This gives maximum performance but requires more setup.
- **Go’s advantage** is the **batteries‑included, zero‑configuration** HTTP endpoint. In 30 seconds you can add `import _ "net/http/pprof"` and get production‑grade profiling. The trade‑off is less precision than Java’s Flight Recorder and fewer analysis views than Chrome DevTools.

---

### 5. Common Mistakes

**1. Exposing pprof without authentication**  
The default handlers reveal stack traces, variable names, and even in‑memory data (via `/debug/pprof/symbol`). In production, always serve pprof on a separate port with network restrictions (e.g., `localhost` only, or sidecar proxy with auth). Never expose it to the public internet.

**2. Forgetting to import pprof with the blank identifier**  
The line `import _ "net/http/pprof"` only works if you use the **default `http.DefaultServeMux`**. If you use a custom mux, you must register the handlers manually:

```go
mux := http.NewServeMux()
mux.HandleFunc("/debug/pprof/", pprof.Index)
mux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
mux.HandleFunc("/debug/pprof/profile", pprof.Profile)
mux.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
mux.HandleFunc("/debug/pprof/trace", pprof.Trace)
```

**3. Running high‑frequency profiling in production**  
Calling `runtime.SetMutexProfileFraction(1)` on a heavily contended mutex will capture every blocking event, causing massive overhead (hundreds of MB of profile data per minute). Always use low fractions (e.g., 100–1000) in production.

**4. Using `go tool pprof` without `-seconds` on a long‑running service**  
Without `?seconds=30`, the profile endpoint returns immediately with a one‑second sample – often too short to capture intermittent issues. Always specify a duration (30–60 seconds) for meaningful data.

**5. Ignoring TLS when profiling remote services**  
If your service uses HTTPS, pprof endpoints will also be HTTPS. Use `go tool pprof https://service:6060/debug/pprof/profile?seconds=30` and handle certificate validation (either with `-insecure` or proper CA setup).

**6. Relying only on live pprof for memory leaks**  
Heap profiles show live objects at a point in time. A slow leak (e.g., 1 KB per minute) may be invisible in a single snapshot. Use `-alloc_space` or `-alloc_objects` to see cumulative allocations over time, or compare two heap profiles taken hours apart:

```bash
$ go tool pprof -base heap1.pb.gz heap2.pb.gz
```

**7. Forgetting `GODEBUG` for additional runtime diagnostics**  
Environment variables like `GODEBUG=gctrace=1` print GC pauses to stderr, and `GODEBUG=schedtrace=1000` prints scheduler activity. These are invaluable for diagnosing latency issues but are often overlooked.

---

### 6. Performance Considerations

**Overhead of pprof endpoints**  

| Profile type | Typical overhead (sampling rate) | Production safe? |
|--------------|----------------------------------|------------------|
| CPU (`/profile`) | 1-3% (100 Hz) | Yes |
| Heap (`/heap`) | <1% (every 512 KB allocation) | Yes |
| Mutex (`/mutex`) | Configurable: 0.5% at fraction=100, >50% at fraction=1 | Fraction >= 100 only |
| Block (`/block`) | Similar to mutex | Fraction >= 1000 recommended |
| Trace (`/trace`) | 10-20% (event logging) | No – for debugging only |

**Why trace is expensive**  
The execution trace logs every scheduling event, channel operation, and system call. Each event requires a nanosecond‑timestamped write to a ring buffer. On a busy server with 10k goroutines, trace overhead can exceed 30% CPU. Only enable traces for a few seconds on a single replica.

**Memory overhead of profiles**  
Profiling data accumulates in internal buffers. CPU profiles cap at a few hundred KB. Heap profiles can grow large if you have millions of allocations; the `pprof` handler streams protobuf‑encoded data without buffering the entire heap into RAM. However, generating a heap profile requires stopping the world (STW) briefly while the runtime walks the heap. For a 10 GB heap, this STW can be 10–50 ms – acceptable for most services but not for sub‑millisecond latency SLAs.

**Comparing with continuous profiling agents**  
Tools like Pyroscope or Parca continuously sample profiles every 10 seconds. This adds 5–10% CPU overhead long‑term. Go’s design favors **on‑demand profiling** – you turn it on only when you suspect a problem. This keeps baseline overhead near zero.

---

### 7. Best Practices

**Production pprof checklist**

1. **Serve on a separate port** (e.g., `:6060`) bound to `localhost` only. Use a sidecar or SSH tunnel for remote access.
2. **Add a minimal auth** – even a simple IP whitelist or a shared secret header prevents accidental exposure.
3. **Enable mutex and block profiling at low fractions** by default in all services:
   ```go
   runtime.SetMutexProfileFraction(100)
   runtime.SetBlockProfileRate(10000)
   ```
   The overhead is negligible (<<0.1%) and you’ll be thankful when a deadlock occurs.

4. **Use `GOTRACEBACK=all`** in production containers. When your process panics, you get goroutine stacks for every goroutine – invaluable for debugging deadlocks.

**Remote debugging with delve**

Sometimes profiles aren’t enough – you need to set breakpoints, inspect variables, or step through code on a live system. Delve supports attaching to a running process:

```bash
# On production host (or via `kubectl exec`)
$ dlv attach 12345 --headless --api-version=2 --listen=:2345

# From your local machine
$ dlv connect prod-host:2345
(dlv) break main.handleRequest
(dlv) continue
```

**But** attaching delve pauses the process (SIGSTOP). This is dangerous on a single‑replica service. Use only on a canary instance or a pod taken out of the load balancer.

**Integrating with OpenTelemetry and distributed tracing**

pprof tells you *what* is slow (e.g., CPU in `json.Unmarshal`). Distributed tracing tells you *why* (e.g., because a downstream database call is slow). Combine them:

- Annotate your spans with `pprof` labels using `runtime/pprof`:
  ```go
  import "runtime/pprof"

  labels := pprof.Labels("endpoint", r.URL.Path, "user_id", userID)
  pprof.Do(ctx, labels, func(ctx context.Context) {
      // your handler logic
  })
  ```
  These labels appear in CPU profiles, letting you break down CPU usage by endpoint or user.

- Use `go.opentelemetry.io/contrib/instrumentation/runtime` to export Go runtime metrics (heap size, GC count, goroutines) as OpenTelemetry metrics. Alert on anomalies.

**Automated profile collection**

For on‑call debugging, script the collection of all relevant profiles:

```bash
#!/bin/bash
PID=12345
OUTDIR=/tmp/debug-$(date +%s)

mkdir $OUTDIR
curl -s "http://localhost:6060/debug/pprof/profile?seconds=30" > $OUTDIR/cpu.prof
curl -s "http://localhost:6060/debug/pprof/heap" > $OUTDIR/heap.prof
curl -s "http://localhost:6060/debug/pprof/goroutine?debug=2" > $OUTDIR/goroutines.txt
curl -s "http://localhost:6060/debug/pprof/trace?seconds=5" > $OUTDIR/trace.out
go tool pprof -png $OUTDIR/cpu.prof > $OUTDIR/cpu.png
```

Run this when an incident starts, then analyze offline.

---

### 8. Examples

**Example 1: A production‑ready service with pprof and safety middleware**

```go
package main

import (
    "context"
    "expvar"
    "fmt"
    "log"
    "net/http"
    _ "net/http/pprof"
    "runtime"
    "time"
)

func main() {
    // Enable profiling with safe fractions
    runtime.SetMutexProfileFraction(100)
    runtime.SetBlockProfileRate(10000)

    // Main application mux
    appMux := http.NewServeMux()
    appMux.HandleFunc("/api", apiHandler)

    // pprof mux – only listen on localhost
    pprofMux := http.NewServeMux()
    // Note: pprof's default handlers are on http.DefaultServeMux,
    // so we manually register them if we want a custom mux.
    pprofMux.HandleFunc("/debug/pprof/", func(w http.ResponseWriter, r *http.Request) {
        // Optional: add a simple secret token check
        if r.Header.Get("X-Debug-Token") != "your-secret" {
            http.Error(w, "forbidden", http.StatusForbidden)
            return
        }
        http.DefaultServeMux.ServeHTTP(w, r)
    })

    // Start pprof server on localhost only
    go func() {
        log.Println("pprof listening on 127.0.0.1:6060")
        if err := http.ListenAndServe("127.0.0.1:6060", pprofMux); err != nil {
            log.Fatal(err)
        }
    }()

    // Start main server
    log.Println("app listening on :8080")
    log.Fatal(http.ListenAndServe(":8080", appMux))
}

func apiHandler(w http.ResponseWriter, r *http.Request) {
    // Simulate work
    time.Sleep(50 * time.Millisecond)
    fmt.Fprintln(w, "OK")
}
```

**Example 2: Capturing a core dump on panic using `GOTRACEBACK`**

```go
// Set environment variable before running: GOTRACEBACK=crash
// This will write a core dump on panic (Linux only)
package main

import (
    "fmt"
    "os"
    "runtime/debug"
)

func main() {
    // Ensure core dumps are enabled (ulimit -c unlimited in shell)
    debug.SetTraceback("crash") // programmatic equivalent of GOTRACEBACK=crash

    go func() {
        defer func() {
            if r := recover(); r != nil {
                fmt.Fprintf(os.Stderr, "panic: %v\n", r)
                debug.WriteHeapDump(os.Stdout.Fd()) // custom heap dump (advanced)
                panic(r) // re-panic to trigger core dump
            }
        }()
        // Force a nil pointer dereference
        var p *int
        *p = 42
    }()

    select {}
}
```

**Example 3: Using `pprof` to compare two heap profiles**

```bash
# Take baseline heap profile
$ curl -s "http://localhost:6060/debug/pprof/heap" > heap1.pb.gz

# Run load test or wait for leak to manifest

# Take second heap profile
$ curl -s "http://localhost:6060/debug/pprof/heap" > heap2.pb.gz

# Compare
$ go tool pprof -base heap1.pb.gz heap2.pb.gz
(pprof) top
# Shows allocations that increased between the two snapshots
(pprof) list MyFunction
```

**Example 4: Custom pprof labels to trace per‑request CPU**

```go
import (
    "runtime/pprof"
    "context"
)

func tracedHandler(w http.ResponseWriter, r *http.Request) {
    labels := pprof.Labels(
        "path", r.URL.Path,
        "method", r.Method,
    )
    ctx := pprof.WithLabels(r.Context(), labels)
    pprof.SetGoroutineLabels(ctx) // important: attaches to current goroutine

    // Do work – CPU samples will include these labels
    processRequest(ctx)
}
```

View labelled profiles with:
```
$ go tool pprof -tagshow=path /path/to/profile
```

---

### 9. Summary & Exercises

**Summary**

- Go’s profiling is built into the runtime and exposed via `net/http/pprof` – no external agents required.
- CPU and heap profiles are low‑overhead (1–5%) and production‑safe. Mutex and block profiles need careful sampling fractions.
- Execution traces are high‑overhead but invaluable for scheduler and channel debugging.
- Core dumps are a last resort; use `dlv core` to analyze them.
- Always secure pprof endpoints, use separate ports, and consider authentication.
- Combine pprof with distributed tracing (OpenTelemetry) and runtime metrics for full observability.

**Exercises**

1. **Build a leaky service and diagnose it**  
   Write a Go HTTP server that leaks memory by appending to a global slice on each request. Expose pprof on `:6060`. Run a load test (e.g., `hey -n 100000 -c 10 http://localhost:8080/leak`). Use `go tool pprof -alloc_space` to identify the leaking line of code. Then fix the leak and verify with another heap profile comparison.

2. **Simulate a deadlock and capture goroutine profiles**  
   Write a program that creates two goroutines, each waiting for the other’s channel send (classic deadlock). Do not use `select` or timeouts. Run the program, then use `curl http://localhost:6060/debug/pprof/goroutine?debug=2` to dump all goroutine stacks. Identify the deadlock by looking for “waiting for channel” cycles. Implement a fix using a third goroutine or a mutex.

3. **Production incident drill**  
   Use a pre‑built “broken” container image (e.g., a Go service with a CPU‑spin bug). Your task: without restarting the container, attach the pprof endpoint (assume it’s exposed on port 6060 but not authenticated), collect a 30‑second CPU profile, and identify the function causing 80% of CPU usage. Then simulate a fix by modifying the code, rebuilding, and deploying – but first, use the profile to prove the fix would work. Write a post‑mortem explaining how the profile guided your root cause analysis.

---

**Aha! Moment:**  
The next time your production service starts melting down, resist the urge to restart it immediately. Instead, reach for `go tool pprof http://service:6060/debug/pprof/profile?seconds=30` – the profile will likely show you exactly what’s burning CPU. Restarting destroys the evidence. Debugging live with pprof turns a panic‑filled incident into a calm, data‑driven investigation.
