## Chapter 38. Case Study: Building a Production Service

This chapter walks through the design and implementation of a production-ready service in Go, end to end. We’ll build **Pulse**, a lightweight observability metrics aggregator that accepts events from applications, stores them, and exposes query APIs. The goal is not to present a single “correct” architecture, but to demonstrate how the Go idioms, performance characteristics, and design philosophy covered in this book merge into a concrete, maintainable system.

We will follow the service from requirements to deployment, stopping at every decision point to ask: *Why this way?* and *What would break if we did the obvious thing?* The chapter is verbose because production systems are.

---

### 1. Basic Usage – The “How” of Pulse

Pulse exposes two HTTP endpoints:
- `POST /ingest` – accepts a batch of metric events (name, value, labels, timestamp) as JSON.
- `GET /query?name=...&from=...&to=...` – returns aggregated values.

The skeleton below captures the essential plumbing: structured logging with `slog`, configuration from environment variables, dependency injection of a store interface, a simple router built with `http.ServeMux`, and graceful shutdown. This is what “basic usage” looks like for a production service in Go—it’s not a tutorial on HTTP servers, but the canonical starting point.

```go
package main

import (
	"context"
	"errors"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/pulse/store"
	"github.com/pulse/handler"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stderr, &slog.HandlerOptions{Level: slog.LevelInfo}))
	slog.SetDefault(logger)

	cfg := configFromEnv()

	db, err := store.NewPostgresStore(context.Background(), cfg.DatabaseURL)
	if err != nil {
		logger.Error("failed to initialize store", "error", err)
		os.Exit(1)
	}
	defer db.Close()

	srv := handler.NewServer(db, logger)

	httpServer := &http.Server{
		Addr:         ":" + cfg.Port,
		Handler:      srv.Routes(),
		ReadTimeout:  5 * time.Second,
		WriteTimeout: 10 * time.Second,
		IdleTimeout:  120 * time.Second,
	}

	// Graceful shutdown
	shutdown := make(chan os.Signal, 1)
	signal.Notify(shutdown, syscall.SIGINT, syscall.SIGTERM)
	go func() {
		<-shutdown
		logger.Info("shutting down")
		ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
		defer cancel()
		if err := httpServer.Shutdown(ctx); err != nil {
			logger.Error("forced shutdown", "error", err)
		}
	}()

	logger.Info("pulse listening", "port", cfg.Port)
	if err := httpServer.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
		logger.Error("server error", "error", err)
		os.Exit(1)
	}
}
```

The `handler.Server` holds the store and logger, implements `Routes()` returning an `http.Handler`. The handler methods are simple, each following the “guard clause” error pattern:

```go
func (s *Server) handleIngest(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
		return
	}
	var batch []store.Metric
	if err := json.NewDecoder(r.Body).Decode(&batch); err != nil {
		s.logger.ErrorContext(r.Context(), "decode ingest", "error", err)
		http.Error(w, "bad request", http.StatusBadRequest)
		return
	}
	if err := s.store.InsertBatch(r.Context(), batch); err != nil {
		s.logger.ErrorContext(r.Context(), "insert batch", "error", err)
		http.Error(w, "internal error", http.StatusInternalServerError)
		return
	}
	w.WriteHeader(http.StatusAccepted)
}
```

This is the “how.” It uses no framework. It leans on the standard library’s battle-tested server, and it isolates the business logic behind small interfaces. Next, we’ll look under the hood at what happens when this code runs.

---

### 2. Under the Hood – The Runtime Engine

When `ListenAndServe` is called, the `net/http` package creates a TCP listener and, for each accepted connection, launches a goroutine to handle it. By default, there is no limit on concurrent connections—Go’s M:N scheduler multiplexes thousands of goroutines onto a handful of OS threads. Each HTTP request inside those connections is also handled in its own goroutine (through the HTTP/1.1 keep-alive loop or HTTP/2 stream). This means our `handleIngest` function runs concurrently with other requests without any explicit thread pool.

**Connection pooling** happens both for incoming HTTP and for our outgoing database connections. `database/sql` maintains a pool of connections (goroutine-safe) with configurable max open/idle sizes. When `InsertBatch` is called, a connection is borrowed from the pool. If all connections are busy and the pool is at max, the call blocks until a connection is returned—no thread is parked, just a goroutine that yields, preserving throughput.

**Garbage collection** runs concurrently in the background. The tri-color mark-and-sweep collector compacts and frees memory, but also introduces latency spikes if we allocate excessively. Each request decodes a JSON body into a slice of `Metric` structs. Those structs and the byte buffer of `json.Decoder` are allocated on the heap (escape analysis sees them escape). If the service handles 10k requests/second with batches of 100 metrics each, we’re creating 1M metric structs per second. The GC must scan all of them. This is where the under-the-hood knowledge pays off: we’ll later discuss how to reduce allocation pressure.

**Stack growth** is seamless. Each goroutine starts with a small stack (~2KB) that grows and shrinks as needed. The `handleIngest` stack will be copied if necessary, transparently. This allows us to write deeply nested calls without worrying about stack overflow, unlike fixed-thread models.

The **scheduler** does work-stealing. If one OS thread has many runnable goroutines and another is idle, it will steal half the queue. This balances the load without manual tuning. Our service benefits from this automatically—each request goroutine is scheduled efficiently.

Understanding this engine allows us to design for predictability: we’ll set `ReadTimeout`/`WriteTimeout` to avoid hanging goroutines, use contexts for cancellation propagation, and tune the database pool to match the concurrency profile.

---

### 3. Why This Design? – The Philosophy

Pulse is deliberately minimalist. Why not use a framework like Gin, Echo, or Fiber? Because the standard library is sufficient and its shortcomings are well-understood. The `http.ServeMux` has no path parameters before Go 1.22, but we can prefix-match and manually extract segments, or we can use the enhanced mux in Go 1.22+. We chose the standard approach because it reduces dependency risk, aligns with the Go team’s long-term support, and forces us to keep routing logic trivial. In a larger service, a lightweight third-party router is acceptable, but we must evaluate the cost.

**Why interfaces for the store?** The `store.MetricStore` interface is defined where it’s consumed (in `handler`), not where it’s implemented. This is the consumer-defined interface principle. It allows us to test the HTTP layer with a mock store without importing the database package, and to swap implementations (Postgres, in-memory for testing) without touching handlers. This is composition and decoupling, not “interface pollution.”

**Why structured logging with `slog`?** Because production diagnostics demand machine-parseable logs. We log errors with full context: `s.logger.ErrorContext(r.Context(), "insert batch", "error", err)`. The context carries request-scoped values (like a trace ID) that the logger attaches automatically if we configure it. This is far superior to `log.Printf`.

**Why graceful shutdown?** Because we must drain in-flight requests before the process exits. The signal handler triggers `Shutdown`, which stops accepting new connections and waits for existing handlers to complete, subject to the context deadline. This is built into `net/http`—no external library needed. This design reflects the Go philosophy: provide the primitives, not the full orchestration framework.

**Why not exceptions?** Errors are values. Every function returns an error. Flow is linear and explicit. This makes the code verbose but predictable. In `handleIngest`, we can see exactly where things can go wrong. No hidden control flow. This matters in production where every error path must be logged and monitored.

---

### 4. Competing Approaches – The Context

Let’s compare Pulse’s architecture with equivalent services in other ecosystems.

**Java / Spring Boot:** A similar service would use `@RestController` with `@RequestBody`, a Spring Data repository, and an application server. The developer experience is rich: dependency injection, declarative transactions, automatic JSON mapping. The price is memory footprint (JVM warm-up, metaspace) and classloader complexity. Error handling might use `@ControllerAdvice` with exceptions, which separates error logic from business code—convenient but less explicit. Concurrency is thread-pool based (Tomcat’s 200 threads by default), which can limit throughput under high I/O unless you switch to reactive stacks (WebFlux). Go’s goroutine model gives you high concurrency without changing programming style.

**Python / FastAPI:** FastAPI with async/await and Pydantic models yields rapid development. The GIL doesn’t hinder I/O-bound workloads much because of asyncio. However, CPU-bound parsing or aggregation can block the event loop; you’d need to run synchronous workers. Error handling often relies on exceptions and `try/except`, which can hide control flow. Deployment often requires uvicorn/gunicorn with multiple workers, adding complexity. Go compiles to a single static binary—simpler ops.

**Rust / Actix-web:** Rust provides memory safety without GC, and Actix gives actor-based concurrency with high performance. The ownership model eliminates data races at compile time. But the learning curve is steep, and development velocity may be slower. Error handling uses `Result` and `?` operator, similar to Go’s explicit style but more ergonomic. Rust’s async story is still maturing and often involves choosing between tokio/async-std. For a metrics aggregator where allocation pressure is manageable, Go’s simplicity wins.

**Node.js / Express:** Single-threaded event loop, callback-based or async/await. Throughput is good for I/O, but heavy JSON parsing can block. The ecosystem is massive, but callback-style error handling and uncaught promise rejections can lead to production surprises. Go’s goroutine-per-request model avoids callback hell and keeps error handling in the same call stack.

Pulse’s design bets on the standard library, low memory overhead, and operational simplicity—areas where Go shines.

---

### 5. Common Mistakes – The Gotchas

Seasoned engineers will encounter these pitfalls when building Go services.

**1. Neglecting request context propagation.** Every handler receives a `context.Context` from the request. This context is cancelled when the client disconnects. If you ignore it and call a database operation without passing `ctx`, you may waste resources on a request that will never be served. Always use `InsertBatch(r.Context(), ...)`. The same applies to HTTP client calls: use `NewRequestWithContext`.

**2. Leaking goroutines in shutdown.** If you start background goroutines (e.g., a periodic cleanup), they must listen on a `<-ctx.Done()` channel or a dedicated stop channel. Otherwise, the process won’t exit cleanly. Use `errgroup` or `sync.WaitGroup` to manage them.

**3. Misconfigured database connection pool.** The defaults for `database/sql` are open-ended: `SetMaxOpenConns(0)` means unlimited. Under spike loads, you can overwhelm the database and exhaust file descriptors. Always set `SetMaxOpenConns` and `SetMaxIdleConns` based on load testing. Also, set `SetConnMaxLifetime` to allow rotation of credentials and avoid stale connections.

**4. Unbounded request bodies.** The JSON decoder will happily read a 1GB payload, causing OOM. Use `http.MaxBytesReader` to limit body size.

**5. Sharing `*sql.DB` across requests without thread-safety awareness?** Actually, `*sql.DB` is safe for concurrent use; the mistake is creating a new pool per request, which leaks connections. Use a single instance, created once.

**6. Overusing global state.** A service with global `var db *sql.DB` makes testing and package isolation hard. Inject dependencies through structs.

**7. Ignoring `WriteTimeout`.** Without it, a slow client can hold a connection indefinitely, tying up a goroutine and potentially leaking memory. Always set read/write timeouts.

**8. Forgetting to close response bodies** when making outbound HTTP calls (not our inbound handler, but if Pulse calls another service). The classic: `resp, err := http.Get(...); defer resp.Body.Close()`. If `err != nil`, `resp` is nil—don’t close nil. Check error first.

**9. Using `panic` for validation errors.** This crashes the whole server (with recovery middleware). Return errors.

**10. Over-engineering with interfaces early.** Don’t define `MetricStore` until you need to swap implementations. In Pulse, we introduced it immediately because testing the handler without a real DB is a clear need.

---

### 6. Performance Considerations – The Cost

Our service must handle sustained load: say 5,000 ingest requests/sec, each with a batch of up to 200 metrics. That’s 1M metrics inserted per second.

**Allocation pressure:** Each decoded `Metric` is a struct of ~64 bytes. 1M allocations/sec is 64 MB/sec of heap allocation. The GC will trigger frequently. To reduce this, we can use `sync.Pool` to recycle slice backings for the batch. Since Go 1.22, `encoding/json` can stream with a `Decoder` and handle multiple JSON objects—we already do that. But the `json.Decoder` still allocates for each token. An alternative for extreme throughput is `json-iterator` or hand-rolled parsing, but that sacrifices readability. Often, the standard library is fast enough; profile before optimizing.

**Database insert efficiency:** Use PostgreSQL’s `COPY` protocol for bulk inserts. `database/sql` supports it via `pq.CopyIn` or `pgx`. We’ll use batch inserts with prepared statements. Each metric becomes a row; 1M rows/sec is beyond a single DB instance capacity. We’d need sharding or a queue (Kafka) in front—but that’s out of scope. The point is: the Go service must be memory-efficient to buffer and batch efficiently. We’ll implement a small in-memory buffer that flushes every second or when a size threshold is reached, using a mutex-protected slice. This reduces database round trips and GC pressure (reuse the slice).

**Goroutine overhead:** 5k goroutines for concurrent requests is trivial (each ~4KB stack, total 20MB). The scheduler will handle them effortlessly. But if we spawn additional goroutines per request for background flushes, we need to manage their lifecycle. Use a single background goroutine that drains a channel.

**Profiling:** We’ll run `pprof` endpoints on a separate port (not publicly exposed) to inspect heap and CPU profiles. A common finding: `fmt.Sprintf` in hot paths allocates. We’ll use `strconv` or `strings.Builder`. The benchmark suite will guide us.

**Latency vs. throughput:** We set `WriteTimeout` to 10s, but our ingest endpoint responds with `202 Accepted` immediately after buffering (not after DB write). The actual persistence happens asynchronously. This decouples client latency from DB latency, at the cost of eventual consistency. That’s a deliberate trade-off for high throughput.

---

### 7. Best Practices – The Idiomatic Way

Production Go services share a set of patterns that Pulse exemplifies:

- **Configuration from environment variables:** Use `os.Getenv` with defaults, not a config file parser unless complexity warrants. Twelve-factor app style.
- **Structured logging with context:** Pass `context.Context` through all I/O calls so that logs, traces, and cancellation are unified.
- **Small, focused packages:** `handler`, `store`, `model` are top-level packages. No deep nesting like `pkg/controller`.
- **Consumer-defined interfaces:** `type MetricStore interface { ... }` lives in `handler` package (or a shared `domain` package if needed), decoupled from `store/postgres`.
- **Middleware as `func(http.Handler) http.Handler`:** We chain logging, recovery, and request ID injection. Avoid middleware that depends on many interfaces; keep them orthogonal.
- **Graceful shutdown:** Use signal handling and `Server.Shutdown()`.
- **Health checks:** A `/healthz` endpoint that pings the database and returns 200 only if everything is healthy. Kubernetes relies on this.
- **Metrics exposure:** A `/metrics` endpoint for Prometheus (using `expvar` or a client library). This is critical for observing performance in production.
- **Repository pattern:** The store interface has methods like `InsertBatch` and `QueryRange`. This keeps SQL out of handlers and makes testing clean.
- **Table-driven tests:** The handler tests feed various input batches and assert HTTP status codes and mock calls. The store tests use a test PostgreSQL instance (via Docker) to verify SQL correctness.
- **Use of `context` for deadlines:** Set a context deadline on DB queries to avoid hanging on a slow DB.

Idiomatic Go services resist the urge to build elaborate frameworks. The code reads like a linear script that can be understood in a single sitting.

---

### 8. Examples – Complete Feature Walkthrough

Let’s add a concrete feature: querying aggregated metric values. The endpoint `GET /query?name=requests_total&from=2026-06-01T00:00:00Z&to=2026-06-02T00:00:00Z` returns the sum, avg, min, max over the period.

First, the store interface (defined in `handler` or a shared `domain`):

```go
type MetricStore interface {
	InsertBatch(ctx context.Context, batch []Metric) error
	QueryAgg(ctx context.Context, name string, from, to time.Time) (AggResult, error)
	Ping(ctx context.Context) error
}
```

The Postgres implementation (package `store`):

```go
type PostgresStore struct {
	db *sql.DB
}

func (s *PostgresStore) QueryAgg(ctx context.Context, name string, from, to time.Time) (AggResult, error) {
	query := `
		SELECT count(*), sum(value), avg(value), min(value), max(value)
		FROM metrics
		WHERE name = $1 AND ts >= $2 AND ts < $3`
	row := s.db.QueryRowContext(ctx, query, name, from, to)
	var r AggResult
	r.Name = name
	r.From, r.To = from, to
	err := row.Scan(&r.Count, &r.Sum, &r.Avg, &r.Min, &r.Max)
	if errors.Is(err, sql.ErrNoRows) {
		return r, nil // empty is valid
	}
	return r, err
}
```

Handler:

```go
func (s *Server) handleQuery(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query().Get("name")
	if name == "" {
		http.Error(w, "name required", http.StatusBadRequest)
		return
	}
	from, err := time.Parse(time.RFC3339, r.URL.Query().Get("from"))
	if err != nil {
		http.Error(w, "invalid from", http.StatusBadRequest)
		return
	}
	to, err := time.Parse(time.RFC3339, r.URL.Query().Get("to"))
	if err != nil {
		http.Error(w, "invalid to", http.StatusBadRequest)
		return
	}
	agg, err := s.store.QueryAgg(r.Context(), name, from, to)
	if err != nil {
		s.logger.ErrorContext(r.Context(), "query agg", "error", err)
		http.Error(w, "internal error", http.StatusInternalServerError)
		return
	}
	w.Header().Set("Content-Type", "application/json")
	if err := json.NewEncoder(w).Encode(agg); err != nil {
		s.logger.ErrorContext(r.Context(), "encode response", "error", err)
	}
}
```

This is complete, but we can optimize the ingest path for high throughput by buffering:

```go
type IngestBuffer struct {
	mu     sync.Mutex
	batch  []Metric
	store  MetricStore
	logger *slog.Logger
	ticker *time.Ticker
}

func (b *IngestBuffer) Add(ctx context.Context, m Metric) error {
	b.mu.Lock()
	defer b.mu.Unlock()
	b.batch = append(b.batch, m)
	if len(b.batch) >= 1000 {
		return b.flush(ctx)
	}
	return nil
}

func (b *IngestBuffer) flush(ctx context.Context) error {
	if len(b.batch) == 0 {
		return nil
	}
	batchCopy := make([]Metric, len(b.batch))
	copy(batchCopy, b.batch)
	b.batch = b.batch[:0]
	go func() {
		if err := b.store.InsertBatch(ctx, batchCopy); err != nil {
			b.logger.ErrorContext(ctx, "buffer flush", "error", err)
		}
	}()
	return nil
}
```

This uses a mutex and copies the slice to release the lock quickly. The actual DB insert is asynchronous, but we must ensure that the `ctx` is not cancelled mid-flight—we could use `context.WithoutCancel(ctx)` if appropriate.

---

### 9. Summary & Exercises

This case study demonstrated how Go’s philosophy translates into a real service: small interfaces, explicit error handling, goroutine-based concurrency without callbacks, and reliance on the standard library. We avoided frameworks, focused on operational concerns (graceful shutdown, structured logging, timeouts), and kept performance characteristics in mind without premature optimization.

**Key takeaways:**
- The `net/http` server, combined with `database/sql`, provides the backbone. Understand their internals to tune them.
- Consumer-defined interfaces decouple packages and ease testing.
- Graceful shutdown is mandatory; `signal.Notify` and `Server.Shutdown` are your friends.
- Profile before optimizing; use `pprof` on a separate admin port.
- Idiomatic Go prefers simplicity: no ORM, no DI container, just clean code.

**Exercises:**

1. **Add authentication middleware.** Extend Pulse to require a bearer token on the `/ingest` endpoint. Implement middleware that validates a JWT and injects a `tenantID` into the request context. Ensure that the downstream handler uses the tenant ID to partition data (multi-tenancy). Write a table-driven test for the middleware.

2. **Implement a gRPC ingest endpoint.** Add a second ingestion path using `google.golang.org/grpc`. Share the same `MetricStore` behind both HTTP and gRPC servers. Handle graceful shutdown for both servers simultaneously. Compare the concurrency model with the HTTP version.

3. **Instrument with distributed tracing.** Integrate OpenTelemetry to propagate trace IDs from ingest to query, and to database calls. Modify the structured logger to include trace IDs automatically. Use `otelhttp` middleware for the HTTP server and `otelgrpc` if you did exercise 2. Finally, set up a local Jaeger instance and verify end-to-end traces.

These challenges reinforce the patterns discussed and build the muscle memory of production Go development.
