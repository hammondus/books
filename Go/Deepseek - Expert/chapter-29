## Chapter 29: HTTP Clients & Servers

The `net/http` package is the backbone of virtually every networked Go service. It is not a minimal wrapper around OS sockets; it is a production-grade, feature-rich HTTP stack that handles connection pooling, HTTP/2, TLS, timeouts, and graceful shutdown without pulling in a single third-party dependency. This chapter dissects the client and server from the inside out, showing how to build fast, reliable, and idiomatic HTTP applications entirely with the standard library.

---

### 1. Basic Usage

**A minimal server** listens on a port, matches a pattern, and writes a response.

```go
package main

import (
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "text/plain")
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("Hello, Go!"))
    })
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

Go 1.22 introduced enhanced pattern matching in the default mux. You can define methods and path variables directly:

```go
http.HandleFunc("GET /items/{id}", func(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    w.Write([]byte("Item ID: " + id))
})
```

The **client** is equally straightforward. The zero-value `http.Client` works, but you should never use it in production.

```go
client := &http.Client{
    Timeout: 10 * time.Second,
}
resp, err := client.Get("https://api.example.com/data")
if err != nil {
    log.Fatal(err)
}
defer resp.Body.Close()
// read body ...
```

Creating a `POST` request with a JSON body and context propagation:

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

req, err := http.NewRequestWithContext(ctx, http.MethodPost,
    "https://api.example.com/items", strings.NewReader(`{"name":"gopher"}`))
if err != nil {
    log.Fatal(err)
}
req.Header.Set("Content-Type", "application/json")

resp, err := client.Do(req)
if err != nil {
    log.Fatal(err)
}
defer resp.Body.Close()
// handle response
```

The pattern is always: create a request with a context, attach a body and headers, execute, and drain the response body completely.

---

### 2. Under the Hood

The `net/http` package is built on three core abstractions: the **server**, the **transport**, and the **handler**. Understanding them explains why Go HTTP is both fast and surprisingly subtle.

**Server** – `http.Server` accepts connections and spawns a goroutine per request. Each connection is read by a `bufio.Reader` and written by a `bufio.Writer` for efficiency. By default, the server supports HTTP/2 via `golang.org/x/net/http2` when TLS is enabled, and it handles keep-alive connections transparently. The server's `ConnContext` hook allows you to attach values (e.g., connection IDs) to the context of every request on that connection.

**Transport (client)** – `http.DefaultTransport` is a shared, global `http.Transport` that manages a pool of idle connections. It dials TCP/TLS connections, caches them by host, and reuses them for subsequent requests. The transport supports:
- HTTP/1.1 keep-alive with pipelining disabled (Go serializes requests over a single connection by default).
- HTTP/2 multiplexing, which allows concurrent requests over a single TCP connection without head-of-line blocking.
- Connection pooling configuration: `MaxIdleConns`, `MaxIdleConnsPerHost`, `IdleConnTimeout`, `MaxConnsPerHost`.

When you execute `client.Do(req)`, the transport checks out an idle connection or creates a new one. On response, the connection is returned to the idle pool if the response body is fully consumed and closed; otherwise, the connection is discarded. This is a critical detail we’ll revisit in Common Mistakes.

**Handler** – `http.Handler` is the interface with a single method `ServeHTTP(http.ResponseWriter, *http.Request)`. Middleware is just a function that takes and returns a `Handler`. The `http.ResponseWriter` is an interface whose concrete implementation buffers the status code and headers until the first `Write`. This means you can set headers after writing the status code only if you understand the buffering semantics (you can’t, but you can use `WriteHeader` explicitly). The default mux (`http.DefaultServeMux`) is an `http.Handler` that holds a radix tree of patterns. Since Go 1.22, the mux supports patterns like `GET /items/{id}` with priorities based on specificity.

**Goroutine per request** – The server creates a goroutine for each incoming request. This model is simple and scales well because goroutines are lightweight, but unbounded goroutine creation under high load can exhaust memory. The server provides `MaxHeaderBytes`, `ReadTimeout`, and `WriteTimeout` to protect against slow clients, but it does not bound the total number of concurrent goroutines—that’s left to the developer or a reverse proxy.

---

### 3. Why This Design?

Go’s HTTP package reflects the language’s core philosophy: a small, composable standard library that enables powerful abstractions without forcing them on you.

**No built-in router framework** – Go deliberately ships with a basic mux that handles simple routing. For years, the community criticized the lack of path parameters. The Go team’s response was measured: first, they ensured the `Handler` interface was flexible enough to support third-party routers (like `gorilla/mux`, `chi`, or `httprouter`). Then, they gradually enhanced the default mux (Go 1.22) when the design space was well-understood. This avoids baking in a single routing paradigm and keeps the standard library small.

**Middleware via function wrapping** – Because `http.Handler` is an interface, middleware is just a higher-order function: `func(http.Handler) http.Handler`. This pattern is astonishingly simple. Compare this to the decorator chains or annotation-based interceptors in Java (JAX-RS filters) or Python (Django middleware classes). Go’s approach demands no reflection, no XML configuration, and no framework lock-in.

**Client-side connection pooling built-in** – The `http.Transport` is a thread-safe, production-grade pool right out of the box. In many languages, HTTP clients are either tied to a heavy framework (e.g., Spring’s `RestTemplate`) or require a separate library (Python’s `requests` with `urllib3`). Go gives you a high-performance client without any third-party dependency. The design pushes the complexity down into the runtime, so you don’t have to configure thread pools or connection managers manually.

**Context integration as a first-class citizen** – The `Request` carries a `context.Context` that controls cancellation and deadlines. The client automatically cancels in-flight requests when the context is done. The server cancels the request context when the client disconnects. This end-to-end cancellation chain is woven into the standard library, making it trivial to build services that respect timeouts and don’t waste resources on abandoned requests. Other ecosystems retrofitted context propagation (Java’s thread locals, Python’s `asyncio` tasks), but Go made it a core part of the request lifecycle.

---

### 4. Competing Approaches

**Java (Servlet API / Spring Boot)** – Java’s HTTP stack is built on the Servlet specification. A servlet runs in a thread pool, and the container manages request dispatching, filters, and lifecycle callbacks. This model is powerful but relies on annotation-driven configuration (`@GetMapping`), thread-local storage for request context, and deep class hierarchies. Go’s single-method interface is a stark contrast: no base classes, no XML, no annotations. The simplicity means you can implement a handler in a single line, but it also means you must build your own routing and middleware composition. Spring Boot provides all that, but at the cost of heavy reflection and slower startup. Go’s approach trades immediate feature completeness for transparency and compile-time safety.

**Python (Flask / FastAPI)** – Python frameworks are built on WSGI or ASGI, which define a calling convention between server and application. Flask decorators map routes to functions, and global request context is managed via thread locals (or context variables). This feels lightweight, but the underlying performance relies on C extensions or event loops. Go’s `net/http` is compiled to native code and runs each request in a goroutine with minimal overhead. There is no GIL; the scheduler multiplexes goroutines onto OS threads. While Python frameworks offer convenience (auto-validation, dependency injection), Go requires more boilerplate but delivers predictable, high-throughput performance without tuning async event loops.

**Node.js (Express)** – Express popularized the middleware chain pattern with `req`, `res`, and `next`. Go’s `http.Handler` wrapping achieves the same effect with pure functions, but without the callback-driven error handling of Node.js. Both platforms handle a request per callback/goroutine, but Node.js’s single-threaded event loop can block on CPU-intensive handlers, requiring careful offloading. Go’s runtime schedules goroutines across multiple OS threads, so a CPU-heavy handler does not stall the entire server. Error handling in Go is explicit via return values; in Express, errors are passed to `next(err)`. The Go approach forces you to think about error paths at every step, which leads to more robust services.

**Rust (hyper / Actix-web)** – Rust’s `hyper` is a low-level HTTP library that provides async building blocks. Frameworks like `Actix-web` or `Axum` build on `tower`’s `Service` trait, which resembles Go’s `Handler` interface but with async functions and zero-cost abstractions. Rust achieves memory safety without a garbage collector, and its type system can encode request validation at compile time. Go’s runtime with a GC sacrifices some raw CPU efficiency and memory predictability for significantly shorter development cycles. Both ecosystems promote composition over inheritance, but Rust’s learning curve is steeper. Go’s HTTP package is accessible to any engineer within a day; Rust’s async HTTP stack requires understanding of lifetimes, traits, and async executors.

---

### 5. Common Mistakes

Even experienced engineers stumble over these subtle behaviors.

**1. Not closing the response body** – This is the most frequent and dangerous mistake. The `http.Client` transport only reuses a connection if the response body is completely read and closed. If you ignore the body, the connection remains open until the client timeout or GC, leaking goroutines and sockets.

```go
resp, _ := client.Get(url)
// LEAK: body not closed, connection not returned to pool
```

Always:

```go
resp, err := client.Get(url)
if err != nil {
    return err
}
defer resp.Body.Close()
io.Copy(io.Discard, resp.Body) // drain if not fully reading
```

If you don’t need the body, you **must** still close it and drain the residual bytes to enable connection reuse.

**2. Using the default `http.Client` without a timeout** – `http.DefaultClient` has no timeout. A slow or malicious server can hold connections open indefinitely, leading to goroutine leaks and resource exhaustion. Always create a client with a `Timeout` or at least a `http.Transport` with dial and response header timeouts.

```go
client := &http.Client{
    Timeout: 30 * time.Second,
    Transport: &http.Transport{
        DialContext: (&net.Dialer{
            Timeout:   5 * time.Second,
            KeepAlive: 30 * time.Second,
        }).DialContext,
        TLSHandshakeTimeout:   5 * time.Second,
        ResponseHeaderTimeout: 5 * time.Second,
    },
}
```

**3. Forgetting context propagation** – The client request must carry a context with a deadline, otherwise a hanging server leaves the client goroutine stranded forever. The server handler must respect the request’s context to abort work when the client disconnects.

```go
select {
case <-ctx.Done():
    return ctx.Err()
case result := <-workChan:
    // process
}
```

**4. Writing to the response after the handler returns** – Because the server handler runs in its own goroutine, spawning a goroutine to write to the `ResponseWriter` after `ServeHTTP` returns is unsafe. The connection may be reused or closed. If you need a fire-and-forget write (like a heartbeat), you must manage the connection lifecycle explicitly.

**5. Incorrect middleware ordering** – Middleware wrapping is semantically a stack. A logging middleware placed before the recovery middleware will not log panics. A timeout middleware that cancels the context must be outer enough to cover the entire request handling but inner enough to not cancel middleware that runs after. The rule: wrap the core handler with the closest concerns, then layer outward.

**6. Ignoring the `Request.Body` ownership** – The handler owns the request body. You must not close it before the handler finishes, and you must not read it concurrently without synchronization. The body is a stream that can be read once. If you need to read it multiple times (e.g., for logging and processing), you must buffer it into a `bytes.Buffer` or `io.TeeReader`.

---

### 6. Performance Considerations

**Connection pooling tuning** – The `http.Transport` defaults are conservative. For high-throughput services, adjust:

- `MaxIdleConns` (default 100) – Total idle connections across all hosts. If your client talks to many backends, increase this to avoid closing and reopening connections.
- `MaxIdleConnsPerHost` (default 2) – Critical! The default of 2 idle connections per host is a bottleneck for services that issue many concurrent requests to the same backend. Increase to match expected concurrency.
- `MaxConnsPerHost` (0 = no limit) – Can be used to prevent overwhelming a backend.
- `IdleConnTimeout` (90s) – Shorten in environments where backends drop idle connections sooner.

**HTTP/2 multiplexing** – When using TLS, Go automatically negotiates HTTP/2. This allows concurrent requests over a single TCP connection, eliminating the per-host connection limit and reducing TLS handshake overhead. HTTP/2 is always enabled with `http.Client` unless you force `TLSNextProto` to disable it. The server also enables HTTP/2 by default with TLS. For high-throughput microservices, HTTP/2 can dramatically reduce latency and connection count.

**Timeouts** – Set a `Timeout` on the client, but also set `ResponseHeaderTimeout` and `ExpectContinueTimeout` to catch slow headers. On the server, `ReadTimeout` and `WriteTimeout` protect against slowloris attacks and stuck clients. `IdleTimeout` on the server allows reuse of connections without holding them forever.

**Zero-copy and buffer reuse** – The server writes headers and body through `bufio.Writer` (4KB buffer). `http.ResponseWriter`’s `Write` copies your bytes into this buffer, then flushes. To reduce allocations, you can implement `io.ReaderFrom` on a custom response writer, but that’s rarely needed. The client reads the response body through a `bufio.Reader`. Use `io.Copy` to stream the body directly to a file or another connection without materializing the entire payload in memory.

**JSON serialization overhead** – The standard `encoding/json` package allocates memory for marshaling. For high-throughput REST APIs, consider using `json.NewEncoder(w).Encode(v)` on the server to stream directly to the response writer. On the client, use `json.NewDecoder(resp.Body).Decode(&v)` to avoid buffering the whole body. Even better, use libraries like `github.com/goccy/go-json` if you need faster, non-reflection-based marshaling. However, the standard library is often sufficient and benefits from the Go team’s optimization efforts.

**Goroutine and memory cost** – Each request spawns a goroutine. Under extreme load (e.g., 100K concurrent connections), the goroutine stack memory (initially 2KB, growing as needed) and the GC pressure from request/response allocations become noticeable. A reverse proxy like Caddy or Envoy can buffer and multiplex connections, reducing the number of goroutines per backend. The `net/http` server itself is not designed for millions of simultaneous idle connections; that’s where a custom event loop with `epoll` or a library like `gnet` would be necessary.

---

### 7. Best Practices

**1. Always propagate and respect context** – The request context carries deadlines, cancellation signals, and request-scoped values (e.g., trace IDs). Pass it to all downstream calls: database queries, internal gRPC, or next HTTP client calls. The server automatically cancels the context when the client disconnects; use `r.Context().Done()` to abort long-running work.

**2. Use custom `http.Transport` and set timeouts** – Do not use `http.DefaultClient` or `http.DefaultTransport` except in trivial scripts. Create an explicit transport configured for your workload. For services, instantiate a shared `http.Client` at startup, not per request.

**3. Close response bodies and drain if necessary** – Make it a habit:

```go
defer func() {
    io.Copy(io.Discard, resp.Body)
    resp.Body.Close()
}()
```

If you fully read the body with `io.ReadAll`, closing is sufficient because reading to EOF causes the transport to return the connection to the idle pool. `io.Copy(io.Discard, ...)` is needed when you only read a part (like checking status code) and want to keep the connection alive.

**4. Structured logging and recovery** – Every server should have a panic recovery middleware that logs the stack trace and returns a 500. Use `slog` with request IDs and context.

```go
func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                buf := make([]byte, 64<<10)
                buf = buf[:runtime.Stack(buf, false)]
                slog.ErrorContext(r.Context(), "panic recovered", "error", rec, "stack", string(buf))
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

**5. Graceful shutdown** – Use `http.Server.Shutdown` or `ListenAndServe` with `Server` struct and handle OS signals to drain in-flight requests.

```go
srv := &http.Server{
    Addr:         ":8080",
    Handler:      mux,
    ReadTimeout:  5 * time.Second,
    WriteTimeout: 10 * time.Second,
    IdleTimeout:  120 * time.Second,
}

go func() {
    if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        log.Fatalf("listen: %s\n", err)
    }
}()

quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit
log.Println("Shutting down server...")

ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
if err := srv.Shutdown(ctx); err != nil {
    log.Fatalf("Server forced to shutdown: %v", err)
}
```

**6. Version your APIs and use structured responses** – For RESTful services, embed version in the URL path (`/v1/items`) or via headers. Return JSON objects with consistent error formats.

**7. Middleware chaining** – Build a helper to chain:

```go
func chainMiddleware(handler http.Handler, middlewares ...func(http.Handler) http.Handler) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        handler = middlewares[i](handler)
    }
    return handler
}
```

Common middlewares: request ID injection, logging, recovery, authentication, CORS, rate limiting.

**8. Client-side retries with backoff** – For transient network errors or 5xx responses, implement a retry `RoundTripper` that retries with exponential backoff and jitter. Use the context deadline to bound total time. Do not retry on non-idempotent methods unless the server supports idempotency keys.

---

### 8. Examples

**Example 1: Production-ready REST server with enhanced routing (Go 1.22+)**

```go
package main

import (
    "log"
    "log/slog"
    "net/http"
    "os"
)

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stderr, nil))
    mux := http.NewServeMux()

    // Items CRUD
    mux.HandleFunc("GET /v1/items", listItems)
    mux.HandleFunc("GET /v1/items/{id}", getItem)
    mux.HandleFunc("POST /v1/items", createItem)

    // Middleware stack
    handler := chainMiddleware(mux,
        recoveryMiddleware,
        loggingMiddleware(logger),
        requestIDMiddleware,
    )

    srv := &http.Server{
        Addr:         ":8080",
        Handler:      handler,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
        IdleTimeout:  120 * time.Second,
    }
    log.Println("Listening on :8080")
    log.Fatal(srv.ListenAndServe())
}

func listItems(w http.ResponseWriter, r *http.Request) {
    // ... fetch items, marshal to JSON
    w.Header().Set("Content-Type", "application/json")
    w.Write([]byte(`[{"id":"1","name":"gopher"}]`))
}

func getItem(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    w.Write([]byte(`{"id":"` + id + `","name":"gopher"}`))
}

func createItem(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusCreated)
    w.Write([]byte(`{"id":"2"}`))
}

func requestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := r.Header.Get("X-Request-Id")
        if id == "" {
            id = "generated-" + randomString(8)
        }
        ctx := context.WithValue(r.Context(), "requestID", id)
        w.Header().Set("X-Request-Id", id)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func loggingMiddleware(logger *slog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            logger.InfoContext(r.Context(), "request started", "method", r.Method, "path", r.URL.Path)
            next.ServeHTTP(w, r)
        })
    }
}
```

**Example 2: Client with retries, timeouts, and connection pool**

```go
func newHTTPClient() *http.Client {
    return &http.Client{
        Timeout: 30 * time.Second,
        Transport: &retryTransport{
            base: &http.Transport{
                MaxIdleConns:        100,
                MaxIdleConnsPerHost: 20,
                IdleConnTimeout:     90 * time.Second,
                TLSHandshakeTimeout: 5 * time.Second,
                DialContext: (&net.Dialer{
                    Timeout:   5 * time.Second,
                    KeepAlive: 30 * time.Second,
                }).DialContext,
            },
            maxRetries: 3,
            backoff:    500 * time.Millisecond,
        },
    }
}

type retryTransport struct {
    base       http.RoundTripper
    maxRetries int
    backoff    time.Duration
}

func (t *retryTransport) RoundTrip(req *http.Request) (*http.Response, error) {
    var lastErr error
    for attempt := 0; attempt <= t.maxRetries; attempt++ {
        resp, err := t.base.RoundTrip(req)
        if err == nil && resp.StatusCode < 500 {
            return resp, nil
        }
        if resp != nil {
            io.Copy(io.Discard, resp.Body)
            resp.Body.Close()
        }
        lastErr = err
        wait := time.Duration(attempt) * t.backoff
        select {
        case <-req.Context().Done():
            return nil, req.Context().Err()
        case <-time.After(wait):
        }
    }
    return nil, fmt.Errorf("max retries exceeded: %w", lastErr)
}
```

**Example 3: Idempotency-aware client usage**

```go
client := newHTTPClient()
req, _ := http.NewRequestWithContext(ctx, http.MethodPost, url, body)
req.Header.Set("Idempotency-Key", uuid.New().String())
resp, err := client.Do(req)
// handle response
```

This pattern prevents duplicate requests when retrying non-idempotent operations.

---

### 9. Summary & Exercises

**Summary**

- The `net/http` server follows a goroutine-per-request model with composable `Handler` middleware.
- The client’s `Transport` provides connection pooling, HTTP/2 multiplexing, and rich configuration; the default client lacks timeouts and must be replaced for production.
- Go’s design emphasizes implicit interfaces, context propagation, and simplicity over framework lock-in. Enhanced routing in Go 1.22 reduces reliance on third-party routers.
- Common pitfalls: unclosed response bodies, missing deadlines, ignored context cancellation, and incorrect middleware ordering.
- Performance tuning revolves around connection pool sizing, timeouts, HTTP/2, and streaming JSON.
- Best practices: always wrap handlers with recovery, inject request IDs, use structured logging, and implement graceful shutdown.

**Exercises**

1. **Build a rate-limited HTTP client:** Implement a custom `http.RoundTripper` that limits the number of concurrent requests per host using a token bucket or semaphore. Ensure it respects the request’s context. Use it to fetch multiple resources concurrently while staying within a given QPS limit.

2. **Design a production-ready middleware chain:** Create a server with at least four middleware components: request ID injection and propagation (using context), structured logging of request duration and status code (using a wrapped `ResponseWriter` that captures the status code), panic recovery with stack trace, and per-IP rate limiting using `golang.org/x/time/rate`. Write a test that verifies the ordering and behavior of the chain under panic and timeout scenarios.

3. **Implement graceful shutdown with active request draining:** Extend the basic `http.Server` to track the number of in-flight requests and wait for them to finish before exiting, but with a hard deadline. Use `sync.WaitGroup` incremented in a middleware and decremented when the handler finishes. Demonstrate that new connections are refused during shutdown, but existing ones complete. Include a test that fires requests and triggers a shutdown concurrently.
