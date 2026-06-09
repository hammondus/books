## Chapter 29: HTTP Clients & Servers

The `net/http` package is the crown jewel of Go’s standard library. It provides a production‑grade HTTP client and server that powers everything from tiny microservices to the infrastructure of companies like Google, Cloudflare, and Uber. Unlike many ecosystems where HTTP is dominated by third‑party frameworks, Go’s standard library delivers a complete, composable, and highly performant foundation. This chapter dissects how to build robust HTTP services and clients, why the package is designed the way it is, and how to avoid the subtle traps that catch even seasoned engineers.

### 1. Basic Usage

The `net/http` package exposes two primary workflows: **serving** HTTP requests and **making** HTTP requests.

#### A Minimal Server

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func helloHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, %s!", r.URL.Path[1:])
}

func main() {
    http.HandleFunc("/", helloHandler)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

- `http.HandleFunc` registers a **handler function** with the default `ServeMux`.
- `http.ListenAndServe` starts a server on port 8080, using the default multiplexer (`nil` means `DefaultServeMux`).

#### A Minimal Client

```go
package main

import (
    "context"
    "fmt"
    "io"
    "log"
    "net/http"
    "time"
)

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    req, err := http.NewRequestWithContext(ctx, "GET", "https://api.example.com/data", nil)
    if err != nil {
        log.Fatal(err)
    }

    client := &http.Client{
        Timeout: 10 * time.Second,
    }
    resp, err := client.Do(req)
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    body, err := io.ReadAll(resp.Body)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Status: %s\nBody: %s\n", resp.Status, body)
}
```

Key points: always use a `Context` with a timeout, set a client‑level timeout as a safety net, and **always close the response body** (even when you don’t read it).

### 2. Under the Hood

The `net/http` package is a masterclass in composable design. Let’s examine its core components.

#### The Handler Interface

Everything in the server revolves around the `http.Handler` interface:

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

- `ResponseWriter` is a `net/http`‑defined interface that controls the HTTP response (headers, status code, body). It writes to the underlying TCP connection.
- `Request` holds all incoming request data (headers, body, URL, etc.).

A `ServeMux` (multiplexer) implements `Handler`. It matches request URLs against registered patterns and forwards to the appropriate handler. When you call `http.HandleFunc`, the package wraps your function as a `HandlerFunc` (a type with a `ServeHTTP` method).

#### Server Internals

- **Per‑request goroutine**: For each accepted connection, the server spawns one goroutine to handle it. That goroutine reads the request, calls the `ServeHTTP` method of your handler chain, and writes the response.
- **Connection management**: The server uses a `net.Listener` to accept TCP connections. It reuses goroutines for keep‑alive connections.
- **Timeouts**: You can set `ReadTimeout`, `WriteTimeout`, and `IdleTimeout` on `http.Server`. `ReadTimeout` covers the duration from accept to request body read; `WriteTimeout` runs from end of request header read to response write.

#### Client Transport

The client side is built around the `RoundTripper` interface:

```go
type RoundTripper interface {
    RoundTrip(*Request) (*Response, error)
}
```

The default implementation is `http.Transport`, which manages **connection pooling** (HTTP keep‑alives), proxies, TLS configuration, and compression. A single `Transport` can be shared across many goroutines. The default `http.DefaultClient` uses `DefaultTransport`, which is a global `Transport` with sensible defaults.

`http.Client` itself is a higher‑level abstraction that adds cookie handling, redirect policies, and timeouts. When you call `client.Do()`, the client:
1. Checks the `Request`’s `Context` for cancellation.
2. Applies the client’s `Timeout` (if any) to the context.
3. Delegates to its `Transport` to perform the actual network I/O.

### 3. Why This Design?

The `net/http` design reflects the Go philosophy of **explicit composition over implicit magic**.

- **Interfaces for flexibility**: By defining `Handler` and `RoundTripper`, the standard library lets you replace the routing (`ServeMux`), the transport, or even the entire protocol stack without changing the calling code. You can write a custom handler that does authentication, logging, or metrics, and it fits seamlessly.
- **No framework required**: The built‑in `ServeMux` supports path patterns (including trailing slashes and sub‑paths) that suffice for many services. You don’t need Gorilla Mux or Chi unless you require regexp routes or middleware chaining sugar – but even those are just `http.Handler` decorators.
- **Explicit error handling**: Every network operation returns an `error`. You must decide how to handle it. There is no hidden exception that unwinds the stack – you log, return an HTTP 500, or degrade gracefully.
- **Control over concurrency**: Because each request runs in its own goroutine, you are free to block, spawn additional goroutines, or use channels. The runtime handles scheduling. No callback hell, no async/await syntax.

The designers explicitly rejected a “magic” framework like Rails or Django inside the standard library. Instead, they provided the lego bricks. This choice has led to a vibrant ecosystem of middleware and routers that all speak `http.Handler`.

### 4. Competing Approaches

| Language / Framework | Philosophy | How Go Differs |
|----------------------|------------|----------------|
| **Node.js / Express** | Event‑driven, single‑threaded. Middleware is a chain of callbacks. Asynchronous operations require callbacks/promises. | Go uses a goroutine per request, which is easier to reason about (synchronous‑looking code). No callback nesting. The standard library `http.Handler` can be wrapped without third‑party tools, whereas Express middleware often requires framework‑specific APIs. |
| **Java / Spring Boot** | Heavy, annotation‑driven, reflection‑heavy. Dependency injection, auto‑configuration. `@RestController`, `@RequestMapping`. | Go’s approach is explicit: you write a function that takes `ResponseWriter` and `Request`. No classpath scanning, no runtime proxy generation. This reduces startup time and memory overhead dramatically. |
| **Python / Flask** | Lightweight but still relies on decorators and global request objects. The server is often not production‑ready (you need gunicorn/uWSGI). | Go’s `net/http` server is production‑ready out of the box. It handles keep‑alives, timeouts, and TLS termination. Python’s GIL limits concurrency; Go’s goroutines scale to tens of thousands of simultaneous connections. |
| **Rust / Actix Web** | Actix uses an actor model with message passing. It can outperform Go in raw throughput but requires managing lifetimes, ownership, and `async` syntax. | Go trades off a few percent of peak performance for dramatically simpler code. You don’t need to think about async executors or pinning. The garbage collector and scheduler handle the rest. |

### 5. Common Mistakes

Even experienced engineers fall into these `net/http` traps.

#### 1. Not Closing the Response Body

```go
resp, _ := http.Get("https://example.com")
// missing defer resp.Body.Close()
body, _ := io.ReadAll(resp.Body)
```

Failure to close the body **leaks a file descriptor** and the associated TCP connection. The default transport will eventually run out of connections. **Always** do:

```go
defer resp.Body.Close()
```

If you don’t need the body, you must still read and close it to enable connection reuse:

```go
io.Copy(io.Discard, resp.Body)
resp.Body.Close()
```

#### 2. Forgetting Timeouts on the Default Client

`http.DefaultClient` has **no timeout**. A request can hang forever. Always create your own `http.Client` with a reasonable `Timeout`, or set a context timeout on each request.

```go
// DANGER: No timeout
resp, err := http.Get("https://slow.example.com")

// SAFE:
client := &http.Client{Timeout: 30 * time.Second}
```

#### 3. Server Timeout Neglect

`http.ListenAndServe` without `ReadTimeout`/`WriteTimeout` leaves the server vulnerable to slow‑loris attacks and resource exhaustion. A client can send headers byte‑by‑byte and hold the connection forever.

```go
srv := &http.Server{
    Addr:         ":8080",
    ReadTimeout:  5 * time.Second,
    WriteTimeout: 10 * time.Second,
    IdleTimeout:  120 * time.Second,
    Handler:      myHandler,
}
log.Fatal(srv.ListenAndServe())
```

#### 4. Sharing the Same Request Across Goroutines

A `http.Request` is **not safe for concurrent modification**. Its `Body` is an `io.ReadCloser` that can be read only once. If you need to process the same request body in multiple goroutines, read it into `[]byte` first and then copy.

#### 5. Ignoring Context Cancellation in Handlers

Long‑running handlers should respect the request’s context. When the client disconnects or the server’s `ReadTimeout` fires, the `r.Context()` is cancelled. Not checking `ctx.Err()` can keep goroutines alive longer than necessary.

```go
func longHandler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    select {
    case <-ctx.Done():
        // client disconnected – stop work
        return
    case result := <-doExpensiveWork(ctx):
        fmt.Fprint(w, result)
    }
}
```

### 6. Performance Considerations

#### Connection Pooling in the Client

The default `Transport` reuses TCP connections (HTTP keep‑alive). Tune it for high throughput:

```go
transport := &http.Transport{
    MaxIdleConns:        100,
    MaxIdleConnsPerHost: 100,
    IdleConnTimeout:     90 * time.Second,
    TLSHandshakeTimeout: 10 * time.Second,
    DisableCompression:  false, // gzip costs CPU but saves bandwidth
}
client := &http.Client{Transport: transport, Timeout: 30 * time.Second}
```

- `MaxIdleConnsPerHost` is critical for services that talk to many downstream hosts. The default is 2 – that’s often too low for high‑volume services.
- `IdleConnTimeout` prevents stale connections from accumulating.

#### Goroutine Overhead Per Request

Each request consumes a goroutine (~2KB stack initially). For 100k concurrent connections, that’s ~200MB of stack – manageable. However, if each handler does heavy CPU work, the scheduler will eventually thrash. Use worker pools (chapters 25, 26) or rate limiting for CPU‑bound handlers.

#### Streaming Responses vs. Buffering

`Write` calls on `http.ResponseWriter` are buffered; the buffer is flushed at the end of the handler or when you call `Flush()` (if the writer is an `http.Flusher`). Streaming large files?

```go
file, _ := os.Open("large.iso")
defer file.Close()
io.Copy(w, file) // w implements io.Writer – this streams block by block
```

Avoid reading the entire file into memory.

#### `defer resp.Body.Close()` Overhead

`defer` adds a tiny (nanosecond) overhead. For extremely latency‑sensitive loops (e.g., thousands of tiny requests), you can call `resp.Body.Close()` directly. But the cost is almost never worth the risk of forgetting to close. Use `defer` as the default.

### 7. Best Practices

#### Custom Server with Timeouts and Graceful Shutdown

```go
func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/api/users", usersHandler)

    srv := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
        IdleTimeout:  120 * time.Second,
    }

    // Graceful shutdown
    go func() {
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    if err := srv.Shutdown(ctx); err != nil {
        log.Fatal("Server forced shutdown:", err)
    }
}
```

#### Middleware Pattern (Function Composition)

Middleware is just a function that takes an `http.Handler` and returns an `http.Handler`.

```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
    })
}

func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("panic: %v", err)
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}

// Chain them
handler := recoveryMiddleware(loggingMiddleware(mux))
```

#### Structured Logging with `slog`

```go
import "log/slog"

func slogMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        slog.Info("request",
            "method", r.Method,
            "path", r.URL.Path,
            "remote_addr", r.RemoteAddr,
        )
        next.ServeHTTP(w, r)
    })
}
```

#### Client Retries with Exponential Backoff

```go
func doWithRetries(ctx context.Context, client *http.Client, req *http.Request, maxRetries int) (*http.Response, error) {
    var lastErr error
    for i := 0; i < maxRetries; i++ {
        resp, err := client.Do(req.WithContext(ctx))
        if err == nil && resp.StatusCode < 500 {
            return resp, nil
        }
        if resp != nil {
            resp.Body.Close()
        }
        if err != nil {
            lastErr = err
        } else {
            lastErr = fmt.Errorf("status %d", resp.StatusCode)
        }

        backoff := time.Duration(1<<uint(i)) * 100 * time.Millisecond // 100ms, 200ms, 400ms...
        select {
        case <-time.After(backoff):
        case <-ctx.Done():
            return nil, ctx.Err()
        }
    }
    return nil, fmt.Errorf("failed after %d retries: %w", maxRetries, lastErr)
}
```

### 8. Examples

#### Example 1: REST API with JSON Handling

```go
type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

var users = map[int]User{
    1: {ID: 1, Name: "Alice"},
}

func getUserHandler(w http.ResponseWriter, r *http.Request) {
    idStr := r.PathValue("id") // Requires Go 1.22+ wildcard pattern: "/users/{id}"
    id, _ := strconv.Atoi(idStr)

    user, ok := users[id]
    if !ok {
        http.Error(w, "user not found", http.StatusNotFound)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("GET /users/{id}", getUserHandler) // Go 1.22+ pattern syntax
    http.ListenAndServe(":8080", mux)
}
```

#### Example 2: Streaming JSON from a Channel

```go
func streamHandler(w http.ResponseWriter, r *http.Request) {
    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "streaming unsupported", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/x-ndjson")
    ch := make(chan User) // real world: feed from database or channel

    go func() {
        defer close(ch)
        for i := 0; i < 10; i++ {
            ch <- User{ID: i, Name: fmt.Sprintf("User%d", i)}
            time.Sleep(500 * time.Millisecond)
        }
    }()

    enc := json.NewEncoder(w)
    for user := range ch {
        if err := enc.Encode(user); err != nil {
            log.Printf("write error: %v", err)
            return
        }
        flusher.Flush()
    }
}
```

### 9. Summary & Exercises

**Summary**

- The `net/http` package provides a complete, composable HTTP server and client based on the `Handler` and `RoundTripper` interfaces.
- Servers default to a `ServeMux` router, but you can replace it with any `http.Handler`.
- Always set timeouts on both server (`ReadTimeout`, `WriteTimeout`) and client (custom `http.Client` with `Timeout`).
- Never forget to close response bodies; use `defer` or risk connection leaks.
- Middleware in Go is simple function composition – no framework magic required.
- Use contexts to propagate cancellation and deadlines through both client and server code.

**Exercises**

1. **Robust HTTP Client**  
   Implement an HTTP client that performs automatic retries for idempotent methods (GET, PUT, DELETE) with exponential backoff, respects context deadlines, and uses a connection pool tuned for 500 concurrent requests to the same host. Write tests that simulate transient failures (e.g., using `httptest.Server` that randomly returns 503).

2. **Graceful Shutdown Middleware**  
   Build a server that supports graceful shutdown *and* provides a `/health` endpoint that returns `200 OK` only when the server is ready to accept traffic. The server should stop accepting new requests upon SIGTERM but allow in‑flight requests to finish. Use `sync.WaitGroup` to track active requests.

3. **Custom Router with Parameter Extraction**  
   Without using third‑party libraries, implement a router that supports route patterns like `/users/{id}/posts/{postID}` and extracts parameters into `r.Context()`. The router must be a drop‑in `http.Handler`. Compare its performance against `http.ServeMux` (Go 1.22+) using benchmarks.
