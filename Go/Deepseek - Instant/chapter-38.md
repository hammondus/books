# Chapter 38: Case Study – Building a Production Service

This chapter synthesises everything we’ve covered into a single, complete production service. We’ll walk through the design, implementation, and deployment of a **distributed task scheduler with a REST API** – a realistic service that handles concurrency, persistence, observability, and graceful shutdown.

The service is called **“Scribe”** – a job queue that accepts scheduled tasks (HTTP POST), stores them in PostgreSQL, and processes them asynchronously using worker goroutines. It exposes endpoints to submit tasks, check status, and list tasks. We’ll build it with the standard library, `slog` for logging, and `database/sql`. No external frameworks – only Go’s built-in capabilities.

---

## Basic Usage

Let’s start with the user’s perspective. After building and running the service, a client can interact with it via curl:

```bash
# Submit a task to run after 5 seconds
curl -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{"payload":"send_welcome_email","run_at":"2025-04-07T12:00:00Z"}'

# Response
{"id":"task_123","status":"scheduled"}
```

```bash
# Get task status
curl http://localhost:8080/tasks/task_123

# Response
{"id":"task_123","status":"completed","result":"email sent","completed_at":"2025-04-07T12:00:01Z"}
```

The service runs a background worker pool that polls the database for due tasks and executes them (here we simulate execution with a delay). The API remains responsive even under heavy task loads.

The code skeleton for `main.go`:

```go
package main

import (
    "context"
    "database/sql"
    "encoding/json"
    "log/slog"
    "net/http"
    "os"
    "os/signal"
    "time"

    _ "github.com/lib/pq"
)

type Task struct {
    ID        string
    Payload   string
    Status    string
    RunAt     time.Time
    Result    *string
    CompletedAt *time.Time
}

func main() {
    // 1. Configuration
    dbURL := os.Getenv("DATABASE_URL")
    if dbURL == "" {
        slog.Error("DATABASE_URL required")
        os.Exit(1)
    }

    // 2. Connect to DB
    db, err := sql.Open("postgres", dbURL)
    if err != nil {
        slog.Error("db open failed", "error", err)
        os.Exit(1)
    }
    defer db.Close()
    db.SetMaxOpenConns(10)
    db.SetMaxIdleConns(5)

    // 3. Create worker pool
    workerPool := NewWorkerPool(db, 4) // 4 concurrent workers
    workerPool.Start(context.Background())

    // 4. HTTP server
    mux := http.NewServeMux()
    handler := NewTaskHandler(db, workerPool)
    mux.HandleFunc("POST /tasks", handler.CreateTask)
    mux.HandleFunc("GET /tasks/{id}", handler.GetTask)

    srv := &http.Server{
        Addr:    ":8080",
        Handler: mux,
    }

    // 5. Graceful shutdown
    go func() {
        slog.Info("starting server", "addr", srv.Addr)
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            slog.Error("server error", "error", err)
            os.Exit(1)
        }
    }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, os.Interrupt)
    <-quit
    slog.Info("shutting down...")

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    workerPool.Stop(ctx) // Wait for workers to finish current tasks
    srv.Shutdown(ctx)
}
```

---

## Under the Hood

This service exercises several critical Go runtime features simultaneously.

### Goroutine Lifecycle & Scheduler

- **HTTP server** runs in its own goroutine (started with `go srv.ListenAndServe`). Each incoming request spawns a temporary goroutine, handled by `net/http`. Under load, the scheduler spreads these across OS threads.
- **Worker pool** contains `N` long‑running goroutines (`Worker.Start`). Each worker blocks on a channel (or in our case, a polling loop with a ticker). When a task is due, the worker executes the user payload. The scheduler cooperatively preempts workers during long‑running CPU tasks (Go 1.14+).
- **Main goroutine** blocks on the `signal` channel, then coordinates shutdown. This pattern ensures that no goroutine outlives its usefulness – a common source of leaks in production.

### Memory Allocations & GC Pressure

Every HTTP request allocates:
- A `Task` struct (on heap – escapes because we pass it to the database and workers).
- JSON decoder buffers (temporary allocations that largely stay on stack thanks to `json.Decoder` reuse).
- Database rows and result sets.

We reduce GC pressure by:
- Reusing buffers where possible (e.g., `bytes.Buffer` pools, though not shown for brevity).
- Using `sql.Stmt.Prepare` to avoid repeated query parsing.
- Limiting the worker pool size so that active goroutines don’t exceed 4–5 per core.

The garbage collector’s default `GOGC=100` works well for this workload. If tasks are very short‑lived, we might tune `GOGC` to 200 to reduce CPU overhead.

### Database Connection Pooling

`database/sql` maintains a connection pool. `SetMaxOpenConns(10)` prevents the service from overwhelming PostgreSQL under peak load. The workers and HTTP handlers compete for the same pool – if all connections are busy, request handlers will block on `db.Query`. This is a form of backpressure that protects the database.

### Context Propagation

Every database call uses a `context` derived from the request context (`ctx := r.Context()`). When a client disconnects or the server shuts down, the context cancels and the database operation aborts. This prevents “orphaned” queries from consuming resources.

---

## Why This Design?

### Worker Pool over Channels‑Based Queue

We deliberately chose a **polling** worker pool (workers `SELECT ... WHERE run_at <= NOW() LIMIT 1`) rather than an in‑memory channel. Why?

- **Persistence**: If the service restarts, tasks are not lost. A channel‑based queue would lose all pending tasks.
- **Scalability**: Multiple instances can share the same database, distributing the load. A channel would be local to one process.
- **Simplicity**: Polling removes the need for a separate job broker (Redis, RabbitMQ). For moderate throughput (hundreds of tasks/sec), this is sufficient.

The trade‑off is latency (tasks may wait up to the poll interval) and database load. For lower latency we could use `LISTEN/NOTIFY` with PostgreSQL, but that adds complexity.

### No ORM, Only `database/sql`

We use raw SQL. An ORM (like GORM) would add reflection, interface boxing, and often hidden allocations. More importantly, ORMs encourage query‑per‑row patterns that lead to N+1 problems. Go’s philosophy favours explicit control: we write the SQL, scan the rows, and handle errors immediately.

### HTTP Multiplexer from Standard Library

`net/http` is powerful enough for most services. We use the new (Go 1.22+) route patterns: `POST /tasks` and `GET /tasks/{id}`. No third‑party router needed. The standard library’s `http.Server` has production‑ready timeouts (we omitted them for brevity but would set `ReadTimeout`, `WriteTimeout`, `IdleTimeout`).

---

## Competing Approaches

| Language / Framework | How It Solves Async Tasks | Go’s Advantage / Disadvantage |
|----------------------|---------------------------|-------------------------------|
| **Python + Flask + Celery** | Celery workers (separate processes) communicate via Redis/RabbitMQ. | Go’s in‑process workers eliminate a message broker, reducing ops complexity. Python’s GIL forces multiple processes for true parallelism – Go uses lightweight threads. |
| **Java + Spring Boot + @Async** | Uses a thread pool executor (`ThreadPoolTaskExecutor`). Tasks are stored in memory – restart loses them. To persist, you need JMS or similar. | Go’s goroutines are far cheaper (KB vs MB stack). Java’s threading model scales poorly beyond thousands of concurrent tasks. |
| **Rust + Actix‑web + Tokio** | Asynchronous runtime with `tokio::spawn`. Persistent queues require external crates (e.g., `lapin` for RabbitMQ). | Rust gives maximum memory control (no GC) and fearless concurrency. But the learning curve is steeper, and you must manage async executors explicitly. Go’s runtime is “batteries included”. |

**Key takeaway:** Go’s combination of a green‑thread scheduler, built‑in HTTP server, and `database/sql` allows you to build a production async task processor in <500 lines of code – without external dependencies. The simplicity is a deliberate design choice: trade absolute performance (Rust) for productivity and operational simplicity.

---

## Common Mistakes (Production Gotchas)

Even seasoned engineers trip on these when moving Go to production.

### 1. Goroutine Leak in the Worker Pool

**Mistake:** Not providing a way for workers to exit on shutdown. The code above includes `WorkerPool.Stop(ctx)` that sets an atomic flag and waits for workers to finish. Without this, `main` exits and the workers are abruptly terminated – leaving tasks in an inconsistent state (e.g., `status='running'` with no one to update).

**Fix:** Always plan shutdown. Use `context.Context` or an atomic boolean plus a `sync.WaitGroup`.

### 2. Unbounded Database Connections

**Mistake:** Not setting `SetMaxOpenConns`. Under peak traffic, the service opens hundreds of connections, exhausting PostgreSQL’s `max_connections` and causing `too many clients` errors.

**Fix:** Tune the pool. For 4 workers + 8 concurrent HTTP handlers, `max_open=20` is reasonable.

### 3. Context Ignored in Workers

**Mistake:** Using `context.Background()` for database queries inside workers. When the service shuts down, workers ignore the cancellation and continue to run.

**Fix:** Pass the shutdown context into each worker’s loop.

### 4. Missing HTTP Timeouts

**Mistake:** Using the default `http.Server` with no timeouts. A slow client can hold a connection (and its goroutine) forever.

**Fix:**
```go
srv := &http.Server{
    Addr:              ":8080",
    ReadTimeout:       5 * time.Second,
    WriteTimeout:      10 * time.Second,
    IdleTimeout:       120 * time.Second,
    ReadHeaderTimeout: 2 * time.Second,
}
```

### 5. Race Conditions in Task Status

If two workers pick the same task (because of a slow transaction), they could process it twice. Our design uses `FOR UPDATE SKIP LOCKED` to avoid this (see Examples section). Without it, duplicate execution is likely.

---

## Performance Considerations

### Throughput & Latency

- **Task submission latency**: <1ms (just database insert).
- **Processing latency**: Poll interval + task execution. With a 1‑second poll and a 100ms task, average latency ~550ms.
- **Maximum throughput** (tasks/sec) = `min(db_write_capacity, worker_pool_size / avg_task_duration)`. With 4 workers and 100ms tasks, theoretical max = 40 tasks/sec.

### Database Load

Polling every second with 4 workers = 4 queries/sec (SELECT ... LIMIT 1). That’s negligible. But if the task table has millions of rows, the `WHERE run_at <= NOW()` index must be tuned:

```sql
CREATE INDEX idx_tasks_run_at_status ON tasks(run_at, status) WHERE status = 'pending';
```

This partial index keeps the index small and fast.

### Memory Footprint

Heap size grows with the number of in‑flight tasks (only those being processed, not queued). Each task holds a few hundred bytes. At 1000 tasks concurrently, ~0.5 MB – trivial. The real memory consumer is the HTTP server’s connection buffers (4KB per idle connection). Set `IdleTimeout` to release them.

### GC Impact

Our design creates few temporary objects per task: one `Task` struct, the SQL rows, and JSON encoders. With `GOGC=100`, the GC will run when heap doubles (~10–20 MB). At 100 tasks/sec, this is fine. If you see high GC CPU, reduce allocations by:
- Reusing `json.Encoder` with a `sync.Pool`.
- Using prepared statements to avoid query string allocations.
- Using `strings.Builder` instead of `+` for concatenation.

---

## Best Practices (Idiomatic Go)

### 1. Structured Logging with `slog`

```go
slog.Info("task processed", 
    "task_id", task.ID, 
    "duration_ms", duration.Milliseconds(),
    "worker_id", w.id)
```

Use `slog.Debug` for verbose output; enable with `-log.level=debug`.

### 2. Health Checks

Add an endpoint for orchestration (Kubernetes readiness/liveness):

```go
mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
    if err := db.PingContext(r.Context()); err != nil {
        w.WriteHeader(http.StatusServiceUnavailable)
        return
    }
    w.WriteHeader(http.StatusOK)
})
```

### 3. Graceful Shutdown with Drain

The `WorkerPool.Stop(ctx)` should:
1. Stop accepting new tasks from the API (set a flag).
2. Wait for currently executing tasks to finish.
3. Close the database connection pool.

### 4. Avoid Global State

`TaskHandler` holds `*sql.DB` and `*WorkerPool` as fields – no package‑level variables. This makes testing easy (swap with mocks).

### 5. Use `any` and Type Assertions Carefully

Our JSON payload is a string; in a real service you might use `json.RawMessage` to delay decoding. Avoid `map[string]interface{}` unless necessary – it allocates and is slow.

---

## Examples

Here is the complete, runnable implementation of the core components. (Assume the database schema is created via migration.)

**Database schema (`schema.sql`):**
```sql
CREATE TABLE IF NOT EXISTS tasks (
    id TEXT PRIMARY KEY,
    payload TEXT NOT NULL,
    status TEXT NOT NULL,
    run_at TIMESTAMP WITH TIME ZONE NOT NULL,
    result TEXT,
    completed_at TIMESTAMP WITH TIME ZONE
);

CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_tasks_run_at_status 
    ON tasks(run_at, status) WHERE status = 'pending';
```

**Task handler (HTTP):**
```go
type TaskHandler struct {
    db     *sql.DB
    pool   *WorkerPool
}

func (h *TaskHandler) CreateTask(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Payload string    `json:"payload"`
        RunAt   time.Time `json:"run_at"`
    }
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    id := "task_" + time.Now().Format("20060102150405") + randomString(6)
    _, err := h.db.ExecContext(r.Context(),
        `INSERT INTO tasks (id, payload, status, run_at)
         VALUES ($1, $2, 'pending', $3)`,
        id, req.Payload, req.RunAt,
    )
    if err != nil {
        slog.Error("db insert", "error", err)
        http.Error(w, "internal error", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusAccepted)
    json.NewEncoder(w).Encode(map[string]string{"id": id, "status": "scheduled"})
}
```

**Worker pool:**
```go
type WorkerPool struct {
    db      *sql.DB
    workers int
    wg      sync.WaitGroup
    stop    atomic.Bool
}

func NewWorkerPool(db *sql.DB, workers int) *WorkerPool {
    return &WorkerPool{db: db, workers: workers}
}

func (p *WorkerPool) Start(ctx context.Context) {
    for i := 0; i < p.workers; i++ {
        p.wg.Add(1)
        go p.worker(ctx, i)
    }
}

func (p *WorkerPool) worker(ctx context.Context, id int) {
    defer p.wg.Done()
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()

    for !p.stop.Load() {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            p.processOneTask(ctx, id)
        }
    }
}

func (p *WorkerPool) processOneTask(ctx context.Context, workerID int) {
    tx, err := p.db.BeginTx(ctx, nil)
    if err != nil {
        slog.Error("begin tx", "error", err)
        return
    }
    defer tx.Rollback()

    var task Task
    // Use SKIP LOCKED to avoid duplicate processing
    row := tx.QueryRowContext(ctx,
        `SELECT id, payload, run_at
         FROM tasks
         WHERE status = 'pending' AND run_at <= NOW()
         ORDER BY run_at
         LIMIT 1
         FOR UPDATE SKIP LOCKED`,
    )
    if err := row.Scan(&task.ID, &task.Payload, &task.RunAt); err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return // no work
        }
        slog.Error("scan task", "error", err)
        return
    }

    // Execute the task (simulate work)
    start := time.Now()
    result := executeTask(task.Payload) // your business logic
    duration := time.Since(start)

    _, err = tx.ExecContext(ctx,
        `UPDATE tasks SET status='completed', result=$1, completed_at=NOW()
         WHERE id=$2`,
        result, task.ID,
    )
    if err != nil {
        slog.Error("update task", "error", err)
        return
    }

    if err := tx.Commit(); err != nil {
        slog.Error("commit tx", "error", err)
        return
    }

    slog.Info("task done", "task_id", task.ID, "worker", workerID, "duration_ms", duration.Milliseconds())
}
```

**Graceful shutdown for worker pool:**
```go
func (p *WorkerPool) Stop(ctx context.Context) {
    p.stop.Store(true)
    done := make(chan struct{})
    go func() {
        p.wg.Wait()
        close(done)
    }()
    select {
    case <-done:
        slog.Info("all workers finished")
    case <-ctx.Done():
        slog.Warn("shutdown timeout, forcing exit")
    }
}
```

---

## Summary & Exercises

We built a production‑ready task scheduler that demonstrates:
- **Concurrency**: Worker pool pattern with graceful shutdown.
- **Persistence**: PostgreSQL for reliable task storage, using `FOR UPDATE SKIP LOCKED`.
- **Observability**: Structured logging with `slog`.
- **Resilience**: Context propagation, timeouts, and connection pooling.
- **Idiomatic Go**: No third‑party frameworks, explicit error handling, composition over inheritance.

The service can be extended incrementally – the hallmark of a well‑factored Go codebase.

### Exercises for the Reader

1. **Add distributed tracing**  
   Integrate OpenTelemetry to trace a request from HTTP POST through database insert to worker execution. Use `context` to propagate trace IDs across goroutines. Measure end‑to‑end latency.

2. **Implement a circuit breaker for the database**  
   If `db.Ping` fails repeatedly, stop accepting new tasks and return `503 Service Unavailable`. Use a library like `sony/gobreaker` or implement a simple one with `atomic` and exponential backoff.

3. **Support task prioritisation**  
   Extend the schema with a `priority INT` column. Modify the worker’s `SELECT` to order by `priority DESC, run_at ASC`. Ensure the index supports this ordering efficiently. How does priority affect the `FOR UPDATE SKIP LOCKED` behaviour?

4. **Build a deployment pipeline**  
   Write a Dockerfile (multi‑stage, using `scratch`) and a Kubernetes Deployment with resource limits. Add a liveness probe that hits `/health`. Configure a HorizontalPodAutoscaler based on queue length (query the number of pending tasks). Measure autoscaling response time.

5. **Convert to an event‑driven architecture**  
   Replace polling with PostgreSQL `LISTEN/NOTIFY`. When a task is inserted, send a notification to wake workers. Compare latency and database load against the polling version. Discuss trade‑offs (complexity, reliability of notifications).

These exercises mirror real production challenges. Completing them will deepen your understanding of Go’s runtime, the standard library’s boundaries, and when to reach for external tools.
