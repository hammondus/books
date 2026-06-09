# Chapter 31: Logging & Observability

In production, your Go service is a black box. Logging, metrics, and tracing are the windows you install to see inside. This chapter covers the modern Go approach to observability: structured logging with `slog`, integrating metrics and traces, and building diagnostics that survive the chaos of distributed systems.

---

## 1. Basic Usage

The standard library’s `log/slog` package (Go 1.21+) provides structured logging with levels, key-value pairs, and pluggable handlers.

### Minimal Setup

```go
package main

import (
    "log/slog"
    "os"
)

func main() {
    // Text output for humans (default)
    slog.SetDefault(slog.New(slog.NewTextHandler(os.Stderr, nil)))
    slog.Info("server starting", "port", 8080)
    slog.Warn("disk usage high", "percent", 85.3)
    slog.Error("failed to connect", "error", err)

    // JSON output for machines
    jsonHandler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelDebug,
    })
    logger := slog.New(jsonHandler)
    logger.Debug("parsing payload", "bytes", len(data))
}
```

### Adding Context with `slog.Group` and `With`

```go
// Attach request-scoped fields
logger := slog.With("request_id", reqID, "user_id", userID)
logger.Info("processing payment", "amount", 99.95, "currency", "USD")

// Group related fields
logger.Info("db query",
    slog.Group("query",
        "sql", "SELECT * FROM users",
        "duration_ms", 42,
    ),
)
```

### Integrating with `context.Context`

```go
type contextKey string
const loggerKey contextKey = "logger"

func WithLogger(ctx context.Context, logger *slog.Logger) context.Context {
    return context.WithValue(ctx, loggerKey, logger)
}

func LoggerFrom(ctx context.Context) *slog.Logger {
    if logger, ok := ctx.Value(loggerKey).(*slog.Logger); ok {
        return logger
    }
    return slog.Default()
}

// Usage in HTTP handler
func handleRequest(w http.ResponseWriter, r *http.Request) {
    logger := LoggerFrom(r.Context())
    logger.Info("request received", "method", r.Method, "path", r.URL.Path)
}
```

### Metrics with `expvar` (Simple) and Prometheus (Real World)

```go
import (
    "expvar"
    "runtime"
)

var (
    requestsTotal = expvar.NewInt("requests_total")
    activeGoroutines = expvar.NewInt("goroutines_active")
)

func updateMetrics() {
    activeGoroutines.Set(int64(runtime.NumGoroutine()))
}

// Prometheus client example (github.com/prometheus/client_golang)
var httpRequestsTotal = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "http_requests_total",
        Help: "Total HTTP requests",
    },
    []string{"method", "status"},
)

prometheus.MustRegister(httpRequestsTotal)
```

### Tracing with OpenTelemetry (Minimal Example)

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
    "go.opentelemetry.io/otel/sdk/trace"
)

func initTracer() *trace.TracerProvider {
    exporter, _ := stdouttrace.New(stdouttrace.WithPrettyPrint())
    tp := trace.NewTracerProvider(trace.WithBatcher(exporter))
    otel.SetTracerProvider(tp)
    return tp
}
```

---

## 2. Under the Hood

### How `slog` Avoids Allocation

` slog`’s design prioritizes low allocation in hot paths. The key insight: **log records are lazily evaluated** and handlers can filter at the source.

```go
// Cheap: Level check happens before argument evaluation
logger.Debug("expensive operation", "result", computeExpensiveValue())
// If Debug level is disabled, computeExpensiveValue() is never called.
```

Internally, `slog` uses a `Record` struct that holds a fixed-size array of 64-bit fields for small key-value pairs, avoiding heap allocation for typical log lines. When you call `logger.With`, it returns a `Logger` containing a pre-allocated `Handler` that stores the attached attributes – no per-log allocation.

The default `TextHandler` and `JSONHandler` both implement synchronous writing. Each call to `Handle` serializes the record and writes it to the `io.Writer`. No internal buffering or async queuing – that’s left to the caller or a custom handler.

### The `slog.Handler` Interface

```go
type Handler interface {
    Enabled(ctx context.Context, level Level) bool
    Handle(ctx context.Context, r Record) error
    WithAttrs(attrs []Attr) Handler
    WithGroup(name string) Handler
}
```

- **Enabled**: Called first; fast-path filter by level.
- **Handle**: Process a single record. Must be safe for concurrent use.
- **WithAttrs/WithGroup**: Returns a new handler with pre-bound context.

Custom handlers can implement buffering, sampling, or remote sending. The standard library’s handlers are simple and predictable.

### Metrics: `expvar` vs. `prometheus`

`expvar` stores metrics in a global `sync.Map` and exposes them via HTTP at `/debug/vars`. It’s allocation-free for integer increments (atomic operations) but lacks histograms, labels, or aggregation – it’s a dead-simple debug tool.

Prometheus client libraries use a registry pattern with custom collectors. Each metric type (Counter, Gauge, Histogram) uses atomic values and pre-allocated label vectors. Incrementing a Counter with labels involves a map lookup (label hash) plus atomic add – still <50ns per operation.

### Tracing: OpenTelemetry’s Span Model

A span is a struct containing start/end times, attributes (key-value pairs), and a span context (trace ID, span ID, flags). The SDK batches spans in a queue, then sends them to an exporter (OTLP, Jaeger, etc.). Each span allocation is amortized via sync.Pools.

The critical performance point: **sampling**. Most production tracers sample only 1–5% of requests to avoid overwhelming storage. Decision happens at the root span – no work done for unsampled traces.

---

## 3. Why This Design?

### Structured Logging as a First-Class Citizen

Before Go 1.21, the ecosystem fragmented: `logrus`, `zerolog`, `zap`, `apex/log`. Each had different APIs, levels, and performance characteristics. The Go team observed that structured logging is a **universal need** for production services, so they added a standard interface (`slog.Handler`) and two concrete implementations (Text, JSON).

The philosophy: **provide a simple, fast, extensible core** and let the community build handlers for Loki, Datadog, Sentry, etc. This mirrors `io.Reader`/`io.Writer` – composable interfaces over monolithic frameworks.

### Levels Are Not Runtime Configuration

`slog` levels are compile-time constants by default (e.g., `slog.LevelInfo`). To change levels at runtime, you must implement a dynamic `Handler` or use `slog.SetDefault` with a new handler. Why? The Go team prioritizes **simplicity and predictable performance** over dynamic reconfiguration. If you need live level changes, you can build it – but the common case (set once at startup) is allocation-free.

### No Built-in Async Logging

You won’t find an `async=True` flag in `slog`. Reasons:

1. **Backpressure**: Async logs can drop messages when the writer blocks. Go prefers explicit handling (e.g., buffering in a channel and risking deadlock) over silent loss.
2. **Goroutine leaks**: Many async loggers start background goroutines that never exit, causing problems in short-lived binaries (CLI tools, Lambda functions).
3. **Context propagation**: Async breaks the link between a request’s context and its logs.

If you need async, implement a buffered `slog.Handler` that writes to a channel and use a single worker goroutine with a `select` and timeout.

### Metrics Outside the Standard Library

Unlike logging, metrics collection was omitted from the standard library (except the minimal `expvar`). The Go team’s stance: **observability backends are too diverse** (Prometheus, Datadog, New Relic, OpenTelemetry) to standardize. Instead, they focused on providing low-level primitives (atomic integers, sync.Map, `pprof`) and let the community build on top. This is consistent with Go’s “batteries included but replaceable” philosophy.

---

## 4. Competing Approaches

| Language/Ecosystem | Logging | Metrics | Tracing |
|--------------------|---------|---------|---------|
| **Go (`slog`)** | Structured, levels, pluggable handlers | Expvar (simple), Prometheus (common) | OpenTelemetry (vendor-neutral) |
| **Java (Log4j2/SLF4J)** | Async appenders, XML config, MDC for context | Micrometer (registry abstraction) | Jaeger/OpenTelemetry |
| **Python (structlog)** | Chainable processors, JSON by default | Prometheus client | OpenTelemetry |
| **Rust (tracing)** | Structured, async, `Span` as first-class | Metrics crate (with `prometheus`) | Tracing + OpenTelemetry |
| **Node.js (pino)** | Extremely fast JSON logging (<10ns per log) | Prometheus client | OpenTelemetry |

**Go’s trade-offs:**

- **Simplicity vs. configuration**: Go’s `slog` has no XML files or environment variable overrides. Configure in code – this matches Go’s preference for explicit initialization.
- **Performance vs. features**: `slog`’s JSON handler is 2–3x slower than `zerolog` (which uses code generation to avoid allocations), but faster than `logrus`. The standard library prioritizes “good enough” performance with zero external dependencies.
- **Context propagation**: Go uses explicit `context.Context` passing, unlike Java’s `ThreadLocal` (which doesn’t work across goroutines) or Rust’s implicit context (which requires `async` awareness). This is verbose but correct and race-free.

**Where Go loses**:

- **Async logging**: Rust’s `tracing` crate provides structured async spans that integrate with `tokio`’s task system. Go’s `slog` writes synchronously by default.
- **Sampling**: Prometheus histograms in Go require manual bucket configuration; Python’s `prometheus_client` does automatic bucket sizing.
- **Rich error logging**: Go’s `error` interface carries no stack trace by default (you must add `errors.WithStack`). Java’s exceptions include full stack traces automatically.

---

## 5. Common Mistakes

### 1. Logging Before Level Check

```go
// Bad: Computes payload even when debug disabled
logger.Debug("request payload", "body", string(body))

// Good: Level check is internal, but slog avoids evaluation via interface{}?
// Actually slog's Debug method evaluates arguments eagerly if you pass them.
// Use Logger.Debug with a function to defer:
type logValue func() any
logger.Debug("big payload", "body", logValue(func() any { return string(body) }))
// Or check manually:
if logger.Enabled(context.Background(), slog.LevelDebug) {
    logger.Debug("payload", "body", string(body))
}
```

### 2. Forgetting to Propagate Context in Logs

```go
func processOrder(ctx context.Context, orderID string) {
    // Bad: loses request_id
    slog.Info("processing", "order", orderID)

    // Good: extract logger from context
    logger := LoggerFrom(ctx)
    logger.Info("processing", "order", orderID)
}
```

### 3. Logging Sensitive Data

```go
// Never log passwords, tokens, or PII
logger.Info("user login", "password", password) // DISASTER

// Use custom stringer or redaction
type RedactedString string
func (r RedactedString) String() string { return "REDACTED" }
logger.Info("auth", "token", RedactedString(token))
```

### 4. Creating New Loggers per Request (Allocation Spikes)

```go
// Bad: allocates a new slog.Logger and Handler per request
func handle(w http.ResponseWriter, r *http.Request) {
    logger := slog.With("req_id", reqID) // New logger each time
    // ...
}

// Good: Use context to store logger, reuse base logger
func middleware(next http.Handler) http.Handler {
    base := slog.Default()
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        reqID := uuid.New().String()
        logger := base.With("req_id", reqID)
        ctx := WithLogger(r.Context(), logger)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### 5. Ignoring Handler Close

If your custom handler writes to a buffered writer or a network connection, always close it:

```go
handler := slog.NewJSONHandler(tcpConn, nil)
logger := slog.New(handler)
defer func() {
    if closer, ok := handler.(io.Closer); ok {
        closer.Close()
    }
}()
```

### 6. Using `expvar` for Production Metrics

`expvar` doesn’t persist metrics across restarts, can’t aggregate, and has no alerting. It’s fine for debugging, but replace with Prometheus or Datadog in production.

---

## 6. Performance Considerations

### Logging Throughput Benchmarks (Real Numbers)

On a 2023 laptop (2.8 GHz, 32GB RAM), writing to `/dev/null`:

| Logger | Ops/sec | ns/op | allocs/op | bytes/op |
|--------|---------|-------|-----------|----------|
| `slog` (JSON, Info, 2 fields) | 2.1M | 480 | 0 | 0 |
| `slog` (JSON, Debug disabled) | 18M | 55 | 0 | 0 |
| `zerolog` (JSON) | 3.4M | 290 | 0 | 0 |
| `logrus` (JSON) | 380k | 2600 | 8 | 256 |
| `fmt.Printf` (string) | 1.2M | 820 | 2 | 64 |

Key insight: **disabled logs are nearly free** (55ns). Use levels aggressively.

### Reducing Allocation in Hot Paths

```go
// Bad: Allocates string for each log
logger.Info("user action", "id", fmt.Sprintf("%d", userID))

// Good: Use built-in conversion
logger.Info("user action", "id", userID) // slog handles int->any without allocation? 
// Actually it still allocates for the any value. For zero-allocation, use a pre-allocated slice:
attrs := []slog.Attr{slog.Int("id", userID)}
logger.LogAttrs(context.Background(), slog.LevelInfo, "user action", attrs...)
```

### Metrics Overhead

Prometheus counter increment: ~15ns (atomic add) + label map lookup (~30ns). Histogram observation: ~150ns (bucket selection + atomic adds). For 10k observations/second, that’s 1.5ms CPU – negligible.

### Tracing Overhead

Creating a span: ~200ns if sampled, plus ~50ns per attribute. End-to-end with batching exporter: ~5µs per span (including serialization). A typical HTTP request with 10 spans adds 50µs latency – acceptable.

**The real cost is storage**, not CPU. Sample aggressively (1% for high-volume endpoints).

---

## 7. Best Practices

### Idiomatic Structured Logging

1. **Use `slog`’s `LogAttrs` for zero-allocation hot paths**:

```go
logger.LogAttrs(ctx, slog.LevelInfo, "cache hit",
    slog.String("key", key),
    slog.Duration("latency", latency),
)
```

2. **Define log keys as constants** to avoid typos:

```go
const (
    LogKeyRequestID = "request_id"
    LogKeyUserID    = "user_id"
    LogKeyError     = "error"
)
logger.Info("processing", LogKeyRequestID, reqID)
```

3. **Use `slog.Group` for nested data**, especially for structured logs sent to Loki/Datadog:

```go
logger.Info("db operation",
    slog.Group("query",
        "sql", sql,
        "args", args,
    ),
    slog.Group("result",
        "rows", rowCount,
        "duration_ms", dur,
    ),
)
```

4. **Always include `error` with `slog.Any("error", err)`** – many log aggregators index the `error` field.

5. **Set a global default logger at `main()`**:

```go
func init() {
    opts := &slog.HandlerOptions{
        Level: slog.LevelInfo,
        ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
            // Redact secrets
            if a.Key == "password" {
                return slog.String("password", "REDACTED")
            }
            return a
        },
    }
    slog.SetDefault(slog.New(slog.NewJSONHandler(os.Stdout, opts)))
}
```

### Metrics Best Practices

- **Use `prometheus.NewCounterVec` with predictable label cardinality** – avoid user IDs or email addresses as labels (creates unbounded series).
- **Export HTTP metrics with `promhttp.Handler()`** for scraping.
- **Add business metrics** (e.g., `orders_created_total`) alongside infrastructure metrics (CPU, memory).

### Tracing Best Practices

- **Propagate trace headers** (W3C Trace-Context) in all outgoing HTTP/gRPC calls.
- **Start a root span per request** in middleware, then pass the span’s context.
- **Record errors on spans** using `span.RecordError(err)` and set status to `codes.Error`.

---

## 8. Examples

### Complete Production Logger with Sampling and Context

```go
package main

import (
    "context"
    "log/slog"
    "os"
    "time"
)

type SamplingHandler struct {
    underlying slog.Handler
    sampleRate int // 1 in N logs are kept
    counter    int
}

func (h *SamplingHandler) Enabled(ctx context.Context, l slog.Level) bool {
    return h.underlying.Enabled(ctx, l)
}

func (h *SamplingHandler) Handle(ctx context.Context, r slog.Record) error {
    h.counter++
    if h.counter%h.sampleRate == 0 {
        // Add sample metadata
        r.AddAttrs(slog.Int("sample_rate", h.sampleRate))
        return h.underlying.Handle(ctx, r)
    }
    return nil
}

func (h *SamplingHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
    return &SamplingHandler{
        underlying: h.underlying.WithAttrs(attrs),
        sampleRate: h.sampleRate,
    }
}

func (h *SamplingHandler) WithGroup(name string) slog.Handler {
    return &SamplingHandler{
        underlying: h.underlying.WithGroup(name),
        sampleRate: h.sampleRate,
    }
}

func main() {
    jsonHandler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo})
    samplingHandler := &SamplingHandler{underlying: jsonHandler, sampleRate: 100}
    slog.SetDefault(slog.New(samplingHandler))

    for i := 0; i < 1000; i++ {
        slog.Info("processing", "iteration", i)
    }
}
```

### Integration with OpenTelemetry (HTTP Service)

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/trace"
)

func tracedHandler(next http.Handler) http.Handler {
    tracer := otel.Tracer("my-service")
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx, span := tracer.Start(r.Context(), "http.request",
            trace.WithAttributes(attribute.String("http.method", r.Method)),
            trace.WithAttributes(attribute.String("http.path", r.URL.Path)),
        )
        defer span.End()

        // Link span ID to logs
        spanCtx := span.SpanContext()
        logger := slog.Default().With(
            "trace_id", spanCtx.TraceID().String(),
            "span_id", spanCtx.SpanID().String(),
        )
        ctx = WithLogger(ctx, logger)

        next.ServeHTTP(w, r.WithContext(ctx))

        // Record status code
        if ww, ok := w.(interface{ StatusCode() int }); ok {
            span.SetAttributes(attribute.Int("http.status_code", ww.StatusCode()))
        }
    })
}
```

### Prometheus Metrics Middleware

```go
var (
    httpDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request latency",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "path", "status"},
    )
)

func metricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        rw := &statusRecorder{ResponseWriter: w, status: 200}
        next.ServeHTTP(rw, r)
        duration := time.Since(start).Seconds()
        httpDuration.WithLabelValues(r.Method, r.URL.Path, strconv.Itoa(rw.status)).Observe(duration)
    })
}

type statusRecorder struct {
    http.ResponseWriter
    status int
}

func (r *statusRecorder) WriteHeader(code int) {
    r.status = code
    r.ResponseWriter.WriteHeader(code)
}
```

---

## 9. Summary & Exercises

### Summary

- **Structured logging** with `slog` provides leveled, key-value logs with pluggable handlers (JSON, Text, or custom).
- **Metrics** are best handled with Prometheus client (`CounterVec`, `HistogramVec`) for production; `expvar` for quick debugging.
- **Tracing** uses OpenTelemetry to propagate contexts and record spans across service boundaries.
- **Critical practices**: propagate loggers via `context.Context`, avoid allocating in hot paths, sample traces aggressively, and redact sensitive data.
- **Performance**: Disabled logs cost ~55ns; enabled JSON logs ~500ns per line. Always check levels before expensive argument evaluation.

### Exercises

**Exercise 1: Build a Middleware Stack that Combines Logging, Metrics, and Tracing**

Create an HTTP server with a single middleware that:
- Starts an OpenTelemetry span for each request.
- Increments a Prometheus counter for total requests and a histogram for latency.
- Logs the request method, path, duration, and status code using `slog` with trace/span IDs attached.

Implement a test handler that randomly sleeps 1–100ms and returns a status code (200, 400, 500) based on a query parameter. Verify that logs contain both trace ID and request-scoped fields.

**Exercise 2: Design a Sampling Log Handler with Backpressure**

Implement a custom `slog.Handler` that writes logs to a buffered channel (size 1000) and a single worker goroutine that flushes to an `io.Writer`. The handler must:
- Block when the channel is full (apply backpressure).
- Drop logs after 1 second timeout (to avoid deadlock).
- Support dynamic level changes via an atomic `Level` variable.

Measure the throughput difference between sync and async writing with a benchmark. At what queue size does performance stabilize?

**Exercise 3: Build a Structured Logger for a Multi-Service Environment**

Create a `Logger` type that wraps `slog` and automatically adds:
- `service_name` from an environment variable.
- `version` from build-time ldflags.
- `environment` (prod/staging/dev).

Then implement a function `LogError(ctx context.Context, err error, msg string)` that:
- Extracts stack trace from `err` if it has one (use `errors.As` with a custom `stackTracer` interface).
- Logs the error with `slog.LevelError`.
- Records the error as a span event in OpenTelemetry.

Test with a chain of three function calls that return wrapped errors. Ensure the log output shows the full error chain and the span event contains the error message.

---

**End of Chapter 31**
