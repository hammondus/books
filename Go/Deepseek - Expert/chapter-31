## Chapter 31: Logging & Observability

A production system is a black box until it speaks. For years, Go projects relied on a patchwork of third‑party logging libraries, each with its own API, performance characteristics, and integration story. With Go 1.21, the standard library finally answered the question: **`log/slog`** — structured, leveled, and extensible. Observability, however, is more than log lines. It spans logging, metrics, and distributed traces, all woven together to give you a real‑time mental model of a running service. This chapter covers `slog` in depth, then builds upward into production diagnostics: how to emit, route, and enrich telemetry so you can debug problems without heroic effort.

### 1. Basic Usage

The entry point is `slog.Logger`. You create one with a **handler** — the component that decides formatting and destination. The zero‑configuration path uses `slog.Default()`, which writes plain‑text key=value pairs to stderr.

```go
package main

import (
	"log/slog"
	"os"
)

func main() {
	// Text handler writes to stderr (default).
	logger := slog.New(slog.NewTextHandler(os.Stderr, nil))
	slog.SetDefault(logger)

	slog.Info("server starting", "port", 8080)
	slog.Warn("disk usage high", "pct", 89.2)
	slog.Error("connection refused", "addr", "db.internal:5432")
}
```

Structured attributes appear as alternating key/value pairs after the message. For machine‑readable output, swap in `slog.NewJSONHandler`:

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo}))
slog.SetDefault(logger)
slog.Info("request", "method", "GET", "path", "/health", "duration_ms", 2)
// {"time":"...","level":"INFO","msg":"request","method":"GET","path":"/health","duration_ms":2}
```

The API deliberately mirrors the old `log` package, but with an extra argument list of key‑value pairs (or `slog.Attr`). You can also attach a `context.Context` to propagate trace identifiers:

```go
slog.InfoContext(ctx, "processing order", "order_id", 1234)
```

Pre‑creating a logger with fixed attributes avoids repetition:

```go
svcLogger := slog.Default().With("service", "checkout", "version", "1.3.0")
svcLogger.Info("payment processed", "amount", 42)
// msg="payment processed" service=checkout version=1.3.0 amount=42
```

The `slog.Level` constants (`Debug`, `Info`, `Warn`, `Error`) follow syslog‑inspired severity order. The handler’s `Level` option gates which records actually reach the handler; messages below the threshold are discarded at the source.

### 2. Under the Hood

`slog` separates **log emission** (the `Logger`) from **log consumption** (the `Handler`). A `Logger` is a lightweight frontend that holds a `Handler` and, optionally, a pre‑loaded set of attributes. When you call `logger.Info("msg", keysAndValues...)`, the logger:

1. Checks if the handler is enabled for `LevelInfo`.
2. Allocates a `slog.Record` — a struct carrying the time, level, message, and a sequence of `Attr`s.
3. Appends the pre‑loaded attributes and the call‑site attributes to the record.
4. Calls `handler.Handle(ctx, record)`.

The `Record` is carefully designed to minimise allocations. Most importantly, `Record` is a value type with internal capacity; the `AddAttrs` method appends to an internal slice without repeated allocation when the pre‑allocated cap is sufficient. If you need to pass a record to a background goroutine (e.g., for asynchronous I/O), you must call `record.Clone()` to decouple it from the caller’s memory.

The `Handler` interface:

```go
type Handler interface {
    Enabled(context.Context, Level) bool
    Handle(context.Context, Record) error
    WithAttrs(attrs []Attr) Handler
    WithGroup(name string) Handler
}
```

`WithAttrs` creates a new handler that always prepends the given attributes (like the `With` method on `Logger`). `WithGroup` opens a named grouping so that subsequent attributes are nested under that key. This enables the “logger inheritance” pattern without expensive runtime reflection.

A `Handler` implementation typically iterates over the `Record`’s attributes (using `Record.Attrs`) and serialises them. The default `commonHandler` (used by both text and JSON handlers) uses `sync.Pool` for internal buffers and avoids reflection by working directly with the `slog.Value` union that encodes Go values into a limited set of kinds (String, Int64, Bool, etc.). This design yields speed close to hand‑rolled optimised loggers, but with a standardised, pluggable interface.

`Level` itself is a `slog.Leveler` interface, allowing custom dynamic level filtering, for example adjusting log volume at runtime through an HTTP endpoint.

### 3. Why This Design?

Go had a standard `log` package since 2009. It was intentionally bare‑bones: no levels, no structure. The community responded with `logrus`, `zap`, `zerolog`, and a dozen other libraries. Why did the Go team wait until 1.21 to add `slog`?

The answer is the Go philosophy: **wait until the right abstraction emerges.** Adding structured logging to the standard library is a commitment to an API forever. The team studied the ecosystem, measured the performance of various designs, and crafted an interface that:

- **Separates emission from representation.** You write log statements once; handlers change format and destination independently. This is the “strategy pattern” baked into the standard library.
- **Embraces the “value” concept.** Rather than force every log call to allocate a `map[string]interface{}`, `slog` uses a lightweight `Attr` list that can often be stack‑allocated. The `Handler` interface receives a concrete `Record`, not an opaque `interface{}`.
- **Stays out of the way of tracing.** `slog` accepts `context.Context` in every call, but does not prescribe how to extract trace information. It’s the handler’s responsibility to pluck span IDs from the context. This keeps `slog` orthogonal to observability standards like OpenTelemetry — you can integrate them without patching `slog` itself.
- **Is minimal but extensible.** You can write a custom handler in under 100 lines. The interface intentionally omits features like asynchronous writing or buffering; those belong in the handler, not the core.

The “Aha!” moment is realising that **structured logging is not about human readability; it’s about signal extraction.** By emitting key=value pairs in a predictable format, you enable grep, jq, and log aggregation systems (Loki, ELK, Datadog) to filter, aggregate, and alert on events that would otherwise be buried in free‑text prose.

### 4. Competing Approaches

**In Go’s own ecosystem**, `zap` (Uber) and `zerolog` (rs) are performance champions. `zap` avoids allocations by using a strongly‑typed `Field` type and manual buffer management, achieving sub‑microsecond log lines. `zerolog` takes a zero‑allocation JSON approach, chaining method calls that accumulate into a shared buffer. `logrus` is older, reflection‑heavy, and slower. How does `slog` compare?

- **Allocation overhead:** `slog` with the default `LogAttrs` pathway (passing `[]slog.Attr` instead of `...any`) achieves near parity with `zap` in many benchmarks. The convenience method `slog.Info` that takes `...any` must wrap each pair into an `Attr`, causing allocations, but the hot path can be optimised.
- **API philosophy:** `zap` provides `SugaredLogger` for quick usage and `Logger` for strict zero‑alloc. `slog` offers a single frontend, with `LogAttrs` as the fast lane. This keeps the API small.
- **Integration:** Because `slog` is in the standard library, any package can log without importing a third‑party dependency. Libraries that used to rely on `log` can now expose a structured logger or accept a `*slog.Logger` as a configuration option, reducing fragmentation.

**Across languages:**

- **Python’s `logging`** module has loggers, handlers, formatters, and filters, all configured via a global hierarchy. It is flexible but notoriously slow and complex. Structured logging requires third‑party formatters.
- **Java (SLF4J/Logback)** uses parameterised messages (e.g., `logger.info("Order {} placed", orderId)`) that defer string formatting until necessary. Structure is often bolted on via MDC (Mapped Diagnostic Context). Go’s approach is simpler: pass key‑value pairs directly; no formatting string templates.
- **Rust’s `tracing`** ecosystem centres on spans and structured events, deeply integrated with async runtimes. Go’s `slog` is more lightweight — spans belong to a separate tracing library — but the structured event model is similar.

Go’s choice to standardise structured logging reflects its overall bias: provide a solid, fast foundation in the standard library, and let the ecosystem build sophisticated observability pipelines on top.

### 5. Common Mistakes

**1. Logging entire objects without thinking.** The statement `slog.Info("request", "req", req)` serialises the entire request struct, which could include headers, body, or other sensitive fields. Always cherry‑pick the fields you need. Use `slog.Group` only when you want nested output.

```go
// Good: explicit attributes
slog.Info("request", "method", r.Method, "path", r.URL.Path)
```

**2. Ignoring levels.** A handler created with `nil` options defaults to `LevelInfo`. Debug logs are silently swallowed. If you need debug output, set `Level: slog.LevelDebug` in `HandlerOptions`. Conversely, leaving the level at `Info` and then logging expensive debug information wastes CPU even though it’s discarded — the `Enabled` check is fast, but argument evaluation still happens unless you guard with `if logger.Enabled(ctx, slog.LevelDebug)`.

**3. Forgetting to use `With()` for contextual fields.** Adding the same `service` and `host` attributes to every log line is tedious and error‑prone. Create a sub‑logger at initialisation time:

```go
logger := slog.Default().With("service", "gateway")
// Now all logs from `logger` include the service attribute.
```

**4. Not propagating `context.Context`.** `slog.Info` does not require a context, but `slog.InfoContext` should be used everywhere a request‑scoped context exists. A handler that extracts a trace ID from the context will only work if you pass the context. If you call `slog.Info` with a context‑rich scenario, that information is lost.

**5. Over‑engineering the handler.** Writing a custom handler that buffers logs and flushes periodically can introduce latency spikes and data loss. Most applications are better served by the standard handlers combined with a sidecar that collects stdout and forwards it. Only build a custom handler when you truly need a synchronous, in‑process backend.

### 6. Performance Considerations

Structured logging adds overhead compared to `fmt.Println`. The cost lies in:

- Allocating the `Record` and its internal `Attr` slice.
- Converting Go values into `slog.Value` (a type switch under the hood).
- Serialising to bytes (JSON encoding or text formatting).

To quantify:

- `slog.Info("msg")` with no attributes allocates a small record and is extremely cheap.
- Each `slog.String("key", value)` passed via `...any` allocates an `Attr` on the heap (because `any` boxing forces escape). The variadic slice `args` itself may also escape.

The escape hatch is `Logger.LogAttrs`, which takes a pre‑built `[]slog.Attr`. Since `slog.String` returns an `Attr` by value, you can inline the attribute construction without extra allocations:

```go
slog.LogAttrs(ctx, slog.LevelInfo, "request completed",
    slog.Int("status", 200),
    slog.Duration("latency", d),
)
```

The compiler can often keep the `Attr` values on the stack, and the slice header is passed by value. This path is similar in performance to `zap`’s `Field` API.

Handler choice matters too. `slog.NewJSONHandler` involves JSON marshaling of each attribute, which is more expensive than the text handler’s simple `key=value` format. For high‑throughput services, consider asynchronous batching in a custom handler (e.g., collect records on a channel and flush in batches), but measure first. The default handlers already use `sync.Pool` for internal byte buffers.

**GC pressure:** Each log line that escapes to the heap increases GC load. Avoid logging inside tight loops; if you must, use `LogAttrs` and pre‑allocate the `[]Attr` outside the loop and reuse it by overwriting fields. Also, keep the handler level high enough that debug logs (which often contain many attributes) are never processed in production.

### 7. Best Practices

1. **Set a global default logger early in `main()`.** Use `slog.SetDefault` so that any library that calls `slog.Info()` uses your handler. Choose JSON for production (machine‑readable) and text for development (human‑readable).

2. **Use `With()` to add static context.** A logger for a subsystem should be created once:

   ```go
   orderLogger := slog.Default().With("component", "order_processor")
   ```

3. **Pass `context.Context` to logging calls.** This enables future handlers to enrich logs with request‑scoped data. Middleware should store a logger in the context using `slog.NewContext` and retrieve it later:

   ```go
   func withLogger(next http.Handler) http.Handler {
       return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
           logger := slog.Default().With("trace_id", traceIDFromCtx(r.Context()))
           ctx := slog.NewContext(r.Context(), logger)
           next.ServeHTTP(w, r.WithContext(ctx))
       })
   }
   ```

   Then a handler down the line can call `slog.InfoContext(ctx, ...)` and the trace ID appears automatically.

4. **Log at the appropriate level.** Use `Debug` for development details, `Info` for normal operational events, `Warn` for anomalies that don’t require immediate action, and `Error` for failures that need human attention. Avoid `Error` for client errors (4xx) — those are part of normal operations; use `Warn` or `Info` with `status` attribute.

5. **Embrace structured attributes as first‑class telemetry.** Instead of `logger.Info("user logged in: " + userID)`, write `logger.Info("user logged in", "user_id", userID)`. This lets you count logins with a simple query: `jq 'select(.msg=="user logged in") | .user_id'`.

6. **Keep handlers simple; offload routing.** Use the standard library handlers to write to stdout or stderr, then let your container orchestrator (Kubernetes, Docker) or a log shipper (Fluentd, Vector) handle aggregation, rotation, and forwarding. This keeps your application stateless and crash‑safe.

### 8. Examples

**Example 1: Production JSON logger with custom level**

```go
func initLogger() *slog.Logger {
    opts := &slog.HandlerOptions{
        Level: slog.LevelInfo,
        // Add source file/line for debugging
        AddSource: true,
    }
    handler := slog.NewJSONHandler(os.Stdout, opts)
    return slog.New(handler)
}

func main() {
    slog.SetDefault(initLogger())
    slog.Info("server started", "listen", ":8080")
}
```

**Example 2: HTTP middleware logging request latency and status**

```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        wrapped := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}
        next.ServeHTTP(wrapped, r)
        level := slog.LevelInfo
        if wrapped.statusCode >= 500 {
            level = slog.LevelError
        }
        slog.LogAttrs(r.Context(), level, "http request",
            slog.String("method", r.Method),
            slog.String("path", r.URL.Path),
            slog.Int("status", wrapped.statusCode),
            slog.Duration("latency", time.Since(start)),
        )
    })
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}
```

**Example 3: Extracting OpenTelemetry trace context and injecting into logs**

Assume you have a `trace.SpanContext` from `go.opentelemetry.io/otel`. A custom handler decorates the base handler to append trace attributes:

```go
type traceHandler struct {
    slog.Handler
}

func (h *traceHandler) Handle(ctx context.Context, r slog.Record) error {
    if sc := trace.SpanContextFromContext(ctx); sc.IsValid() {
        r.AddAttrs(
            slog.String("trace_id", sc.TraceID().String()),
            slog.String("span_id", sc.SpanID().String()),
        )
    }
    return h.Handler.Handle(ctx, r)
}

// Usage
baseHandler := slog.NewJSONHandler(os.Stdout, nil)
logger := slog.New(&traceHandler{Handler: baseHandler})
```

This pattern keeps trace injection non‑intrusive. The middleware stores the tracing‑enabled context, and the custom handler enriches logs transparently.

### 9. Summary & Exercises

Structured logging with `slog` is now the idiomatic way to emit telemetry in Go. It separates emission from consumption, minimises allocations with `LogAttrs`, and integrates naturally with `context.Context` for distributed tracing. When combined with metrics and traces, it transforms logs from a textual stream into a queryable, correlated dataset — the foundation of a debuggable production system.

**Exercises**

1. **Asynchronous log handler**
   Build a custom `slog.Handler` that receives `Record` objects on a buffered channel. A separate goroutine drains the channel and writes them to an underlying `slog.Handler`. Ensure that the handler correctly clones records and that the draining goroutine is shut down cleanly. Measure throughput with the `testing/benchmark` package and compare to a synchronous handler.

2. **Intelligent HTTP logging middleware**
   Extend the middleware in Example 2 to:
   - Log request body size and response body size (by wrapping `http.ResponseWriter` to count bytes written).
   - Log at `Warn` level for client errors (4xx) and at `Error` level for server errors (5xx).
   - Include a `request_id` from a header or generate one if missing, and add it to both the response header and the log context.

3. **Tracing‑aware structured logging**
   Integrate `slog` with OpenTelemetry by writing a `Handler` that, in addition to trace/span IDs, extracts baggage items (user‑defined key‑value pairs propagated through context) and appends them to each log record. Write a test that creates a span, stores baggage, calls a function that logs, and verifies the output contains the baggage attributes.
