## Chapter 17: Panic and Recover

When a program reaches a state that violates a fundamental invariant—a nil pointer dereference, an out-of-bounds slice access, a failed mandatory initialization—there are only two responsible options: halt immediately with a loud diagnostic, or attempt a controlled shutdown. Go provides `panic` and `recover` precisely for these moments. They are not a general error-handling mechanism; they are emergency circuit breakers. In this chapter, we will examine what `panic` actually does, how `recover` works inside `defer`, why the design deliberately constrains recovery, and how to use these tools without undermining the reliability of our systems.

---

### 1. Basic Usage

A `panic` stops normal execution of the current function immediately. After that, any deferred functions in the current goroutine run in LIFO order. If one of those deferred functions calls `recover`, the panic sequence halts and execution resumes normally after the function that recovered. If no deferred function recovers, the program crashes with a stack trace.

```go
package main

import "fmt"

func mayPanic(flag bool) {
    if flag {
        panic("something went badly wrong")
    }
    fmt.Println("mayPanic completed normally")
}

func main() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered from:", r)
        }
    }()

    fmt.Println("before mayPanic")
    mayPanic(true)
    fmt.Println("after mayPanic") // this line never executes
}
```

Output:
```
before mayPanic
Recovered from: something went badly wrong
```

Notice that `after mayPanic` is never printed. The `mayPanic` function terminates at `panic`, the deferred function in `main` runs, and `main` returns cleanly. The rest of the program continues after `main` (here the program ends). The `recover` built-in returns the value passed to `panic`, which can be of any type. If no panic is in progress, `recover` returns `nil`.

`recover` **must be called directly inside a deferred function**. This will not work:

```go
func faultyRecover() {
    defer myRecover() // myRecover calls recover inside it
}

func myRecover() {
    if r := recover(); r != nil {
        fmt.Println("recovered", r)
    }
}
```

The call to `recover` happens inside `myRecover`, not in the body of the deferred function literal. The runtime checks that `recover` is called from a function that is directly deferred; nested function calls break the recovery. The correct pattern is:

```go
defer func() {
    if r := recover(); r != nil {
        // handle
    }
}()
```

You can also assign the deferred function to a variable before deferring, but the crucial point is that `recover` must be lexically inside the deferred function’s body.

`recover` only stops the currently active panic. If multiple panics occur during unwinding (a deferred function panics while a panic is already in progress), `recover` captures the most recent one. The earlier panic is not lost—the runtime chains them—but a single `recover` can only extract the latest. We will explore this further under the hood.

---

### 2. Under the Hood

A goroutine maintains a linked list of deferred function calls. Each `defer` statement appends a record that contains the function pointer and any captured arguments. The list grows from newest to oldest. When a function returns normally, deferred calls execute in the reverse order of their creation.

When `panic` is called, the runtime sets a flag on the goroutine’s stack (specifically, a `_panic` structure is pushed onto the goroutine’s panic stack). Normal execution of the current function halts immediately—no more statements are executed. The runtime then starts popping and invoking deferred functions from the current function’s defer list. After all defers of the panicking function have run, if the panic hasn’t been recovered, the runtime moves up to the caller, pops its deferred functions, and so on. This process is called **panicking** or unwinding.

When a deferred function calls `recover`, the runtime checks whether the goroutine is currently in a panicking state. If so, `recover` retrieves the argument of the innermost active panic (the most recent `_panic` structure), marks that panic as recovered, and returns that value. The panicking function’s frame is discarded—it never resumes. The deferred function that recovered returns, and the function that contained the deferred call then returns normally to its caller, as if it had never panicked. Execution continues from the point just after the call to the function that recovered.

Example: function `A` calls `B`. `B` defers a recover and then calls `C`, which panics. `C`’s defers run (none in this case), then `B`’s defers run. The recover catches the panic. `B` returns to `A`, which continues normally. `C`’s stack frame is gone.

If a deferred function panics while a panic is already unwinding, the runtime pushes another `_panic` on top of the current one, linking them. The unwinding continues with the new panic. A `recover` will retrieve the value of the topmost panic. If the program crashes, the runtime prints all panics in the chain, showing the stack trace for each. This provides a complete picture of cascading failures.

The runtime also records a stack trace at the point of the original panic, which is printed on crash or captured via `debug.Stack()` inside a recover block. The trace includes all goroutines if the crash is fatal.

A few implementation details worth knowing:

- The defer list is part of the goroutine’s stack frame chain; popping a frame restores the previous list. This makes defer reasonably fast in the common case.
- `recover` is a compiler intrinsic. It is not a normal function call; the compiler ensures it can only appear inside a deferred function.
- Panics can be triggered by the runtime itself for fatal conditions like concurrent map writes, out-of-memory, or stack overflow. Some of these are unrecoverable (e.g., fatal runtime errors kill the process immediately without running defers).

---

### 3. Why This Design?

Go’s approach to `panic`/`recover` is a deliberate departure from the exception systems common in many languages. The design philosophy rests on three observations:

1. **True unrecoverable errors should be fatal by default.** Most errors in a program are expected: a network timeout, a missing file, invalid user input. These belong in the normal control flow and are best handled by explicit error returns. Exceptions blur the line between expected and unexpected, encouraging developers to throw and catch for flow control—a pattern that hides failure paths and makes code difficult to reason about.

2. **Recovery is an emergency escape, not a control structure.** By restricting `recover` to the body of a deferred function, Go enforces that you cannot simply “catch and continue” from the exact point of failure. The panicking function is terminated. This prevents the common anti-pattern of resuming execution after a state-corrupting error, which can turn a small bug into silent data corruption. If you recover, you do so from a well-defined boundary, usually the top-level caller or a goroutine’s entry point.

3. **Cleanup should be explicit and predictable.** `defer` provides a deterministic, stack-based mechanism for resource cleanup. Instead of scattering `finally` blocks or scope guards, Go centralizes cleanup logic in one place. The fact that defers run during panics guarantees that resources are released even on failure, but only if the developer has structured their defers correctly.

The Go team considered and rejected try/catch/finally. Rob Pike’s “Errors are values” essay and the Go FAQ explain that exceptions make it too easy to ignore errors. By making error handling explicit and using `panic`/`recover` only for truly exceptional programmer mistakes (invariant violations, nil pointer dereferences, index out of range), Go keeps the normal path clear and forces the developer to confront failure conditions at every step.

The ability to recover at all exists mainly to allow long-running systems—like HTTP servers—to survive panics in individual request handlers. The `net/http` package recovers from panics in handler goroutines, logs the error and stack trace, and returns a 500 response, preventing one buggy handler from bringing down the entire server. This pattern is intentional: the server can remain available, but the faulty handler is terminated, and the error is not swallowed.

---

### 4. Competing Approaches

**Java / C# / C++ / Python / JavaScript** – These languages treat exceptions as a general error propagation mechanism. Any function can throw, and any caller can catch. While this reduces visible error-handling boilerplate, it introduces implicit control flow paths. A function’s signature may not reveal all the exceptions it can throw (checked exceptions in Java attempt to address this, but are often circumvented). Performance-wise, throwing an exception is often expensive due to stack unwinding and object construction. In Go, returning an error value is a plain function return, which is essentially free.

Go’s `defer` + `recover` is simpler but less powerful: no type filtering (you must type-assert the recovered value), no resumption semantics, no `finally` (though `defer` covers that). The absence of “catch by type” encourages a single recovery block per boundary, which is often enough for logging and graceful failure.

**Rust** – Uses the `Result` type and the `?` operator to propagate errors without exceptions. Panicking exists for unrecoverable bugs (like array out-of-bounds), similar to Go’s `panic`, but Rust’s panics can optionally be caught with `std::panic::catch_unwind` at thread boundaries. Rust’s `panic` is also unsafe to catch in some contexts due to unwinding and exception safety. The philosophy aligns closely with Go: errors are values; panics are for bugs.

**C** – Has no built-in exception handling. Errors are communicated via return codes or `errno`. Cleanup requires manual `goto` chains. Go’s `defer` dramatically simplifies this pattern while maintaining explicit error returns.

**Zig** – Zig treats errors as values with `error` union types and the `try` keyword. It has no hidden control flow, similar to Go’s explicit `if err != nil`. Panics in Zig correspond to `@panic`, which is unrecoverable and triggers a stack trace. Zig’s approach is arguably even more strict: no recovery mechanism at all.

Go’s `panic`/`recover` occupies a pragmatic middle ground: strong discouragement of using panic for normal errors, but an escape hatch for well-defined recovery at goroutine boundaries, preserving system availability without compromising explicitness.

---

### 5. Common Mistakes

**Mistake 1: Using panic for regular error handling**
```go
func parseConfig(path string) *Config {
    data, err := os.ReadFile(path)
    if err != nil {
        panic(err) // Don't do this.
    }
    // ...
}
```
This forces callers to recover, discards the structured error, and crashes the program if the caller doesn’t wrap every call in a recovery. The idiomatic approach is to return `nil, fmt.Errorf("reading config: %w", err)` and let the caller decide.

**Mistake 2: Swallowing panics silently**
```go
defer func() {
    recover() // The value is completely ignored.
}()
```
If a nil pointer dereference occurs, the program will continue with potentially corrupted state, no log, no stack trace. Always capture, log, and if possible return an error or restart the goroutine.

**Mistake 3: Calling recover inside a nested function**
```go
func myHandler() {
    defer func() {
        helper() // helper calls recover() - ineffective!
    }()
    // ...
}
```
The call to `recover` must be lexically inside the deferred function literal. It cannot be hidden inside another function.

**Mistake 4: Thinking recover resumes after the panic point**
Recovery does not go back and re-execute the line after the panic. The panicking function is terminated. Consider:
```go
func process(data []int) {
    defer func() { recover() }()
    for i := range data {
        if data[i] == 0 {
            panic("zero value")
        }
        // process non-zero
    }
}
```
After the first zero, the entire `process` function stops. The loop does not continue. The caller of `process` continues, and `process` returns zero values for any return parameters. If you need to skip bad items, use a regular error check.

**Mistake 5: Panicking with `nil`**
`panic(nil)` causes `recover` to return `nil`, which is indistinguishable from no panic. This can silently conceal a crash. Use a descriptive value, ideally an error or string.

**Mistake 6: Forgetting that an unrecoved panic in any goroutine crashes the whole program**
A panic in a goroutine that is not the main goroutine will crash the entire process unless it is recovered. Always start goroutines with a recovery wrapper if a crash is unacceptable.

---

### 6. Performance Considerations

`panic` and `recover` are not designed for performance. In the common case—where no panic occurs—the overhead of `defer` is the only cost. Starting with Go 1.13, simple defers that call a function with no arguments can be inlined by the compiler, reducing overhead to almost zero. However, even with that, a deferred function that contains a `recover` check adds a small amount of code bloat and a runtime check on each return.

When a panic does occur, the runtime must walk the defer chain, execute each deferred function, and unwind the stack. This involves multiple allocations (for the `_panic` structure), runtime checks, and context switches. The cost is on the order of microseconds to milliseconds depending on the depth of the call stack and the complexity of the deferred functions. In contrast, returning an error value is a single copy of an interface value on the stack—essentially free.

Profiling a tight loop that panics and recovers will show significant overhead. Use panic only for truly exceptional paths (initialization failures, programmer errors). Never place `panic` on a hot code path. The same goes for `recover`: wrapping every function call in a recover-defer block would add unnecessary overhead and obscure the logic. Instead, recover at the boundary of a goroutine or request.

GC pressure: Panics themselves do not normally cause extra garbage unless the panic value is heap-allocated, but that is negligible. The main cost is the execution of deferred functions, which may perform allocations.

The runtime’s panic path also interacts with the scheduler: during unwinding, the goroutine is in a special state that can affect preemption. The details are not usually a concern, but it reinforces that panics are not a fast-path mechanism.

---

### 7. Best Practices

**1. Use panic only for invariant violations**
A program should panic when it reaches a state that the programmer believes is impossible: an unreachable `default` in a type switch, a negative value where only positive makes sense, a mandatory initialization function that fails at startup. These are bugs that must be fixed, not runtime conditions to handle.

```go
func mustConnect(dsn string) *sql.DB {
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        panic(fmt.Sprintf("failed to open database: %v", err))
    }
    return db
}
```
At program startup, this is acceptable. Inside a request handler, it is not.

**2. Recover only at goroutine boundaries**
In long-running services, start each goroutine with a recovery wrapper that logs the panic and optionally restarts the goroutine. The `net/http` package does this for you per request. For custom goroutines, use a helper:

```go
func safeGo(f func()) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                log.Printf("goroutine panicked: %v\n%s", r, debug.Stack())
            }
        }()
        f()
    }()
}
```

**3. Always log the stack trace**
When recovering from an unexpected panic, capture and log the stack trace using `debug.Stack()`. This gives you the exact location of the panic, which is invaluable for debugging.

**4. Type-assert the recovered value and handle known types**
If your package defines sentinel panic values for internal use, you can selectively recover them and let others propagate.

```go
defer func() {
    if r := recover(); r != nil {
        if err, ok := r.(MyRecoverableError); ok {
            // handle gracefully
        } else {
            panic(r) // re-panic unknown
        }
    }
}()
```
Be very careful with this pattern; it’s easy to hide bugs.

**5. Don’t recover from runtime panics that leave state corrupted**
A panic from the runtime due to a concurrent map write or a nil pointer dereference may indicate a serious bug that has corrupted memory. Recovering and continuing may lead to worse failures. Often the safest response is to crash and restart. If you do recover at a request boundary, make sure the handler’s state is fully reset. For example, an HTTP server using standard `net/http` recovers and finishes the response; subsequent requests use fresh state.

**6. Prefer returning errors for anything a caller might want to handle**
If a function can fail for reasons that are expected (network down, file not found, invalid input), return an error. This makes the failure part of the contract and gives callers the choice of how to proceed.

**7. Avoid `panic` in library code**
Libraries should almost never call `panic`. They have no idea about the application’s recovery strategy. Instead, return errors. The exception is truly unrecoverable internal invariant failures, but even those should be returned as errors if possible.

---

### 8. Examples

**Recovering in an HTTP middleware**

A common production pattern: wrap HTTP handlers to recover panics and return a 500 status.

```go
func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("panic in handler: %v\n%s", err, debug.Stack())
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

The server never crashes because of a buggy handler. The client gets a generic error, and the logs contain the full context.

**Selective recovery with re-panic**

Imagine a transaction helper that panics on purpose to rollback, then the caller recovers.

```go
var ErrRollback = errors.New("rollback requested")

func withTransaction(db *sql.DB, fn func(tx *sql.Tx) error) (err error) {
    tx, err := db.Begin()
    if err != nil {
        return err
    }
    defer func() {
        if p := recover(); p != nil {
            if e, ok := p.(error); ok && errors.Is(e, ErrRollback) {
                err = tx.Rollback()
            } else {
                panic(p) // re-panic unexpected
            }
        } else if err != nil {
            tx.Rollback()
        } else {
            err = tx.Commit()
        }
    }()
    err = fn(tx)
    return
}
```

The `fn` can call `panic(ErrRollback)` to force a rollback. The deferred function recovers only that specific error, rolls back, and returns the rollback error. Any other panic propagates.

**Recovery from a goroutine**

```go
func main() {
    done := make(chan struct{})
    go safeGo(func() {
        // work that might panic
        var ptr *int
        *ptr = 42 // nil pointer dereference
    })
    <-done
}
```

`safeGo` logs the panic and stack trace, then the goroutine ends. The program doesn’t crash.

**Demonstration of panic chain**

```go
func main() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered:", r)
        }
    }()
    defer func() {
        panic("second panic")
    }()
    panic("first panic")
}
```

Output:
```
Recovered: second panic
```
The first panic is chained; `recover` only retrieves the top one. If you didn’t recover, the crash output would show both.

---

### 9. Summary & Exercises

`panic` and `recover` are the Go mechanism for handling truly catastrophic, unrecoverable conditions—bugs and invariant violations, not expected errors. `panic` halts the current function and unwinds the stack, executing deferred functions along the way. `recover`, when called directly inside a deferred function, stops the unwinding and allows the containing function to return normally. This design reinforces the explicit error handling philosophy, prevents silent resumption of corrupted state, and provides just enough flexibility to keep long-running services alive.

Key takeaways:
- Use `panic` only for programmer errors, never for normal control flow.
- `recover` must be inside a deferred function’s body, not a nested call.
- Always log the panic value and stack trace when recovering.
- Recover at goroutine or request boundaries; never swallow unknown panics.
- Performance penalty is high; keep panics out of hot paths.

**Exercises**

1. **Goroutine Safety Wrapper**
   Write a function `GoSafely(fn func() error)` that starts a new goroutine executing `fn`. If `fn` returns an error, log it. If `fn` panics, recover, log the stack trace, and return. The wrapper should ensure that a panicking goroutine never crashes the whole program. Use it to execute a set of tasks concurrently and observe the behavior.

2. **HTTP Recovery Middleware with Request ID**
   Build upon the middleware example: add request ID extraction (from context) and ensure that the panic log includes the request ID. Write a test that sends a request to a handler that intentionally panics, and verify that the client receives a 500 status and the log contains the panic and request ID.

3. **Panic Chain Behavior**
   Write a program that demonstrates the panic chain: function A defers a recover, function B defers a panic, function C panics. Experiment with recover returning only the latest panic. Then, add another recover in B’s defer and see if you can capture the original panic. Explain why the runtime chains panics and the implications for recovery.
