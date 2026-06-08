## Chapter 34: Debugging Production Systems

Production debugging is not about stepping through code in an IDE. It is about observing a running system—often under heavy load, with limited access, and no room for trial-and-error—and extracting the one piece of information that explains the anomaly. Go’s philosophy of “batteries included, but not forced upon you” manifests here: the standard library ships with `net/http/pprof` and `runtime/pprof`, the runtime emits controllable core dumps, and `context.Context` enables distributed tracing without a framework. This chapter equips you to use these tools systematically when every second of downtime costs money.

---

### 1. Basic Usage

Start with the simplest, runnable primitives. Importing `net/http/pprof` and listening on a separate port is the canonical “zero-config” start. For core dumps, setting an environment variable changes the runtime behavior.

**Exposing pprof endpoints**

```go
package main

import (
	"log"
	"net/http"
	_ "net/http/pprof" // side-effect import: registers handlers on DefaultServeMux
	"time"
)

func main() {
	// Production rule: never expose debug on public interfaces.
	// Bind to localhost or a dedicated network interface.
	debugAddr := "localhost:6060"
	go func() {
		log.Fatal(http.ListenAndServe(debugAddr, nil))
	}()

	// Simulate work
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(50 * time.Millisecond)
		w.Write([]byte("ok"))
	})
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

The paths `/debug/pprof/` offer profiles (`profile`, `heap`, `goroutine`, `threadcreate`, `block`, `mutex`). A 30-second CPU profile is fetched with:

```
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
```

**Programmatic profile capture**

For automated diagnostics, use `runtime/pprof` directly:

```go
import "runtime/pprof"

func captureHeapProfile(path string) error {
	f, err := os.Create(path)
	if err != nil {
		return err
	}
	defer f.Close()
	return pprof.WriteHeapProfile(f)
}
```

**Enabling core dumps**

Set `GOTRACEBACK=crash` before starting your process. When the program receives SIGQUIT or panics, it writes a core file containing the full memory image and all goroutine stacks. The core can then be inspected with `dlv core <binary> <corefile>` or Go’s own tools.

```bash
GOTRACEBACK=crash ./myapp
# In another terminal:
kill -QUIT <pid>
```

**Injecting distributed tracing with OpenTelemetry**

Minimal setup using context propagation:

```go
import (
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
)

func initTracer() (*sdktrace.TracerProvider, error) {
	exporter, err := stdouttrace.New(stdouttrace.WithPrettyPrint())
	if err != nil {
		return nil, err
	}
	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter),
		sdktrace.WithSampler(sdktrace.AlwaysSample()), // careful: only for dev
	)
	otel.SetTracerProvider(tp)
	return tp, nil
}
```

---

### 2. Under the Hood

Understanding the mechanics prevents misinterpretation.

**CPU profiling** is signal-driven. The runtime sets up a SIGPROF handler that fires at 100 Hz by default. At each tick, the profiler records the program counter of the currently running goroutine. After the sampling period, these samples are aggregated into a call graph. The overhead is proportional to the sample rate; at 100 Hz it is typically below 5%.

**Heap profiling** uses a different strategy: a subset of allocations is recorded based on `runtime.MemProfileRate`. The default rate is 512 KiB; every allocation of at least 512 KiB, or every allocation that pushes the cumulative size past a 512 KiB boundary, is sampled. This means many small allocations may never appear in the profile. The profiler tracks stack traces at allocation time, not at GC. Therefore, `alloc_space` and `inuse_space` represent allocated bytes and live bytes, respectively. The latter depends on whether the object is still reachable when the profile is taken.

**Goroutine profiles** are a full snapshot: the runtime walks all goroutines and captures their entire stack. This is a stop-the-world operation, but it completes in microseconds on modern systems. The profile is identical to a panic dump but obtained on demand.

**Core dumps** rely on the ELF core format on Linux (mach-o on macOS, minidump on Windows). When `GOTRACEBACK=crash` is set, the runtime hooks into fatal signals and writes additional notes containing Go-specific data: all goroutine stacks, the heap bounds, and the runtime’s internal structures. A standard debugger like `gdb` or `dlv` can parse these notes to reconstruct the Go state. Without this setting, the runtime only prints stack traces to stderr.

**Distributed tracing** leverages the `context.Context` as the carrier for `trace.SpanContext`. When an HTTP request arrives, the propagator extracts the trace ID and span ID from headers (typically W3C `traceparent`). The Go tracing library instruments `net/http` to create a server span automatically. The `context` flows through goroutines, ensuring children are linked to the same trace. The overhead is dominated by span allocation and the exporter; batch processing and sampling keep it manageable.

**Why Go embeds these tools in the standard library?** Because runtime observability is not an afterthought. The runtime already collects scheduling, allocation, and GC statistics. Exposing them to the developer costs almost nothing in binary size and provides a universal debugging vocabulary across all Go programs.

---

### 3. Why This Design?

The alternatives would have been to rely entirely on external tooling (like `perf` or `gdb`) or to provide a heavy-weight agent separate from the binary. Go’s choice to integrate `pprof` directly reflects the “one binary” ethos: you deploy a single artifact, and that artifact can self-diagnose.

- **No separate agent**: Java Mission Control or JMX require a connection layer and JVM cooperation. Go’s `pprof` is just an HTTP handler. Any tool that speaks HTTP can pull profiles.
- **Signal-based CPU profiling** avoids the need for compiler instrumentation. It samples what the hardware is actually executing, giving realistic profiles even for optimized code.
- **Core dumps with Go-aware metadata** turn a generic Unix mechanism into a high-level debugging tool. Instead of manually cross-referencing addresses with symbol tables, you can run `dlv core` and immediately see goroutines, channels, and slices.
- **Tracing via `context.Context`** is a deliberate rejection of thread-local storage (TLS) or ambient contexts. By making the trace explicit in the context argument, Go forces developers to propagate tracing information through the call stack, which aligns with the “share memory by communicating” principle. It avoids invisible side effects and makes tracing testable.

This design embodies **simplicity over complexity**: a handful of standard paths, a few environment variables, and zero configuration files. It also enables composition; you can serve pprof behind your own mux, add authentication middleware, or capture profiles to a file and send them to an external collector.

---

### 4. Competing Approaches

**Java**: JMX (Java Management Extensions) and JFR (Java Flight Recorder) offer deep JVM-level profiling with low overhead. However, they require a JMX agent or API, and the data format is Java-specific. Go’s pprof is simpler but less detailed in some dimensions (no object allocation heatmaps by type without heap profiling, no lock contention timelines without block profiling). Java’s tracing often uses ThreadLocal for context propagation, which can lead to silent context loss in async code. Go’s explicit `context` solves that.

**Python**: Tools like `py-spy` and `faulthandler` mimic Go’s signal-based profiling and traceback dumps. However, Python’s GIL and C extensions make sampling less reliable. Go’s runtime is cooperative, so every preemption point guarantees a consistent view. Python lacks a standard core dump format that understands the interpreter state; Cores only capture the C stack.

**C++/Rust**: They rely on external profilers (`perf`, `VTune`, `tracy`). Core dumps are OS-level and require debug info (DWARF) to interpret. Go’s static linking and custom notes make core dumps instantly useful without separate debug symbol packages. Rust’s `tracing` crate provides span propagation, but the ecosystem is fragmented; Go’s `context.Context` is universal.

**Node.js**: The `--inspect` flag and Chrome DevTools offer a rich debugging experience, but only during development. Production profiling often requires a separate sidecar. Go’s on-demand profiles are designed from the start for production use.

Go’s approach is a middle path: not as feature-rich as JFR, but more production-accessible than raw `gdb` and easier to integrate than tools requiring separate binaries.

---

### 5. Common Mistakes

- **Exposing `net/http/pprof` on the public internet**: The debug endpoints include heap and goroutine dumps that may leak sensitive data. Always bind to `localhost` or a private management interface, or add middleware for authentication.

- **Enabling block/mutex profiles in production without understanding overhead**: `runtime.SetBlockProfileRate(1)` records every blocking event, which can drastically increase CPU and memory usage. Use a rate like 1000 (events per billion nanoseconds) for production if needed.

- **Misinterpreting `alloc_space` vs. `inuse_space`**: A heap profile in `alloc_space` mode shows total allocations since process start, not live memory. If you suspect a memory leak, use `inuse_space` (the default in `go tool pprof`). The `--inuse_space` flag clarifies intent.

- **Forgetting to import `_ "net/http/pprof"` but using custom mux**: The side-effect import only registers on `http.DefaultServeMux`. If you use a custom mux, you must explicitly mount the handlers. Use `pprof.Handler("profile")` and friends.

- **Core dumps with insufficient ulimit**: The shell’s `ulimit -c` may be 0, disabling core creation. Set `ulimit -c unlimited` before starting the process, or call `syscall.Setrlimit` from the application.

- **Heavy tracing that crashes the process**: Enabling `AlwaysSample()` on a high-RPS service generates an enormous number of spans, which can exhaust memory and push latency. Always use a probability sampler (e.g., 0.1%) in production, with remote parent sampling decisions honored.

- **Using `GOTRACEBACK=crash` for panic recovery**: Core dumps can be huge (the process’s full virtual memory). Use `GOTRACEBACK=single` or `all` for less critical services, and only `crash` when you have infrastructure to collect and analyze cores.

---

### 6. Performance Considerations

Every debugging feature has a cost; the key is understanding the trade-off and only paying it when necessary.

**CPU profiling**: At 100 Hz, the cost is around 5% in CPU-bound workloads. It scales linearly with the sample rate; a 500 Hz profile may cost 25%. The profiler stops sampling when the buffer fills, so shorter profiling periods (5–10 seconds) are effective and reduce average overhead.

**Heap profiling**: The default `MemProfileRate` (512 KiB) means roughly 1 sample per 512 KiB of allocated memory. The profiler must acquire a mutex for each sample, so extremely allocation-heavy code (e.g., millions of tiny objects per second) can slow down noticeably. Lowering the rate (e.g., `runtime.MemProfileRate = 0` to disable, or set to a higher value) reduces contention. If you enable `inuse_space` profiling, note that the runtime must traverse the live heap at every profile request, which is a short STW pause.

**Goroutine profiles**: A goroutine profile stop-the-world period is O(number of goroutines) but typically < 1 ms for 100,000 goroutines. The main cost is the latency spike, not throughput.

**Block and mutex profiles**: Both are off by default because they can be expensive. A block profile records the stack and duration every time a goroutine blocks on a channel or sync primitive. If you set `runtime.SetBlockProfileRate(1)`, every single block event triggers a stack capture and time recording, potentially causing significant overhead. Use `10000` (10 µs resolution) as a starting point.

**Tracing overhead**: A typical OpenTelemetry implementation adds a few microseconds per span creation plus the cost of context propagation. Batch span processing reduces export overhead. With a 1% sampling rate, the overhead on a 100k RPS service might be 1–2% CPU and a few megabytes of memory for the exporter queue. Avoid custom attribute collection in hot paths.

**Core dumps**: Writing a core dump stalls the process until the entire memory image is written to disk. For a process using 8 GB of RAM, this can take seconds. Use `GOTRACEBACK=crash` only when you can tolerate a brief outage, and consider compressing the core file externally (with `systemd-coredump` or similar).

**Design principle**: provide a side-channel (separate port, local socket) for debugging that does not interfere with the primary request path. The overhead then only applies during active diagnostics.

---

### 7. Best Practices

**Separate debug listener with authentication**

```go
mux := http.NewServeMux()
mux.HandleFunc("/debug/pprof/", pprof.Index)
mux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
// ... etc.

authMiddleware := func(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		token := r.Header.Get("X-Debug-Token")
		if token != os.Getenv("DEBUG_TOKEN") {
			http.Error(w, "forbidden", http.StatusForbidden)
			return
		}
		next.ServeHTTP(w, r)
	})
}
debugServer := &http.Server{
	Addr:    "localhost:6060",
	Handler: authMiddleware(mux),
}
```

**Automated heap dump on memory pressure**

Use a goroutine that monitors `runtime.MemStats` and triggers a profile when `HeapInuse` exceeds a threshold.

```go
func monitorAndDump(threshold uint64) {
	ticker := time.NewTicker(10 * time.Second)
	defer ticker.Stop()
	for range ticker.C {
		var m runtime.MemStats
		runtime.ReadMemStats(&m)
		if m.HeapInuse > threshold {
			filename := fmt.Sprintf("heap_%d.pprof", time.Now().Unix())
			captureHeapProfile(filename)
			slog.Warn("Heap threshold exceeded, captured profile", "file", filename)
		}
	}
}
```

**Graceful core dump on SIGQUIT**

Instead of only relying on environment variables, register a signal handler that dumps stacks and then re-raises SIGQUIT to generate a core with `GOTRACEBACK=crash`.

```go
go func() {
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGQUIT)
	<-sigCh
	// Dump all goroutines to stderr before coring
	pprof.Lookup("goroutine").WriteTo(os.Stderr, 2)
	// Reset SIGQUIT to default and re-send to self
	signal.Reset(syscall.SIGQUIT)
	syscall.Kill(syscall.Getpid(), syscall.SIGQUIT)
}()
```

**Tracing: wrap, don’t replace**

Wrap `http.Handler`s with an OpenTelemetry middleware, and pass `context.Context` through all layers. Use `otel.Tracer` to create child spans for database calls, external HTTP requests, and expensive computations.

```go
func otelMiddleware(next http.Handler) http.Handler {
	return otelhttp.NewHandler(next, "my-service")
}
```

**Health vs. debug endpoints**: Do not combine health checks with pprof on the same path. Health checks must be lightweight and expose minimal information; pprof must be protected. Use separate listeners.

**Production sampling strategy**: Set `OTEL_TRACES_SAMPLER=parentbased_traceidratio` and `OTEL_TRACES_SAMPLER_ARG=0.01` to sample 1% of traces without breaking parent-child relationships.

**Security**: Never expose pprof over the internet. If you need remote access, use an SSH tunnel or a VPN. A common pattern is to run a sidecar that pulls profiles from localhost and pushes them to an internal storage.

---

### 8. Examples

**Complete production-ready microservice with separate debug server and tracing**

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"runtime"
	"runtime/pprof"
	"syscall"
	"time"

	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
)

func initTracer() (*sdktrace.TracerProvider, error) {
	exporter, err := stdouttrace.New(stdouttrace.WithPrettyPrint())
	if err != nil {
		return nil, err
	}
	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter),
		sdktrace.WithSampler(sdktrace.TraceIDRatioBased(0.01)),
	)
	otel.SetTracerProvider(tp)
	return tp, nil
}

func main() {
	if err := run(); err != nil {
		log.Fatal(err)
	}
}

func run() error {
	tp, err := initTracer()
	if err != nil {
		return fmt.Errorf("init tracer: %w", err)
	}
	defer tp.Shutdown(context.Background())

	// Main HTTP server
	mainMux := http.NewServeMux()
	mainMux.HandleFunc("/api", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(20 * time.Millisecond)
		w.Write([]byte(`{"status":"ok"}`))
	})
	mainHandler := otelhttp.NewHandler(mainMux, "main-service")
	mainServer := &http.Server{Addr: ":8080", Handler: mainHandler}

	// Debug server on separate port with auth
	debugMux := http.NewServeMux()
	debugMux.HandleFunc("/debug/pprof/", pprof.Index)
	debugMux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
	debugMux.HandleFunc("/debug/pprof/profile", pprof.Profile)
	debugMux.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
	debugMux.HandleFunc("/debug/pprof/trace", pprof.Trace)
	authDebug := func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			if r.Header.Get("X-Debug-Token") != os.Getenv("DEBUG_TOKEN") {
				http.Error(w, "forbidden", http.StatusForbidden)
				return
			}
			next.ServeHTTP(w, r)
		})
	}
	debugServer := &http.Server{Addr: "localhost:6060", Handler: authDebug(debugMux)}

	// Start servers
	go func() { log.Fatal(mainServer.ListenAndServe()) }()
	go func() { log.Fatal(debugServer.ListenAndServe()) }()

	// Graceful shutdown with stack dump on SIGQUIT
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT)
	sig := <-sigCh
	log.Printf("Received signal %v, shutting down", sig)
	if sig == syscall.SIGQUIT {
		pprof.Lookup("goroutine").WriteTo(os.Stderr, 2)
		runtime.GC() // update heap profile before dump
		f, _ := os.Create("final_heap.pprof")
		if f != nil {
			defer f.Close()
			pprof.WriteHeapProfile(f)
		}
	}

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	mainServer.Shutdown(ctx)
	debugServer.Shutdown(ctx)
	return nil
}
```

This example demonstrates separation, authentication, tracing integration, and automated profiling on shutdown signals.

---

### 9. Summary & Exercises

Debugging production Go systems hinges on a handful of integrated tools: pprof for profiling and heap analysis, core dumps for post-mortem state, and distributed tracing for cross-service latency decomposition. The design choices—signal-based CPU profiling, sampling-based heap profiles, and context-driven tracing—reflect Go’s commitment to minimal overhead and maximum utility. Missteps like exposing debug endpoints or misinterpreting memory profiles are common, but easily avoided with proper network isolation, authentication, and a clear understanding of sampling semantics.

**Exercises**

1. **Auto-dump on memory threshold**: Write a service that monitors `runtime.ReadMemStats` every second. When `HeapInuse` exceeds 80% of `HeapSys`, it automatically writes a heap profile and a goroutine profile to a timestamped file. Use a rolling buffer to keep only the last 10 dumps. Test by simulating a leak (e.g., a global slice that keeps growing).

2. **Debug endpoint with one-time token**: Implement an HTTP debug server that generates a single-use token via a separate endpoint (e.g., `POST /debug/token`). The token is valid for 5 minutes and only usable once, after which it expires. Use a combination of `sync.Map` and background cleanup goroutine. Ensure that pprof handlers are inaccessible without a valid token.

3. **Distributed bottleneck trace**: Deploy two simple Go services (e.g., `frontend` and `backend`). Instrument them with OpenTelemetry using the `otelhttp` middleware. Configure sampling at 5%. Introduce an artificial delay in `backend` for specific parameters. Use a trace viewer (like Jaeger) to locate the slow request and identify the bottleneck function. Then add custom spans to the `backend` function to show the latency breakdown. Write a brief report on how the trace led you to the root cause.
