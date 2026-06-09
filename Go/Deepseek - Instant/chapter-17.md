## Chapter 17: Panic and Recover

Panic and recover are Go’s mechanism for handling truly exceptional situations—those where the program cannot continue safely. Unlike exceptions in languages like Java or Python, Go’s panic is deliberately minimal, coarse-grained, and designed to signal *programmer* errors (e.g., out‑of‑bounds slice access, nil pointer dereference) or unrecoverable environmental failures (e.g., unable to bind to a required port). Recover gives you a way to intercept a panic, clean up, and optionally restart a component without tearing down the entire process.

This chapter assumes you already understand error handling via `error` return values (Chapter 16). Here we explore what happens when returning an error is impossible or meaningless.

---

### 1. Basic Usage

Panic is a built‑in function that stops the normal flow of control, begins unwinding the stack, and runs all deferred functions. Recover is another built‑in that regains control of a panicking goroutine—**but it only works inside a deferred function**.

```go
package main

import (
	"fmt"
	"os"
)

// Simulate a function that panics on invalid input.
func process(positive int) {
	if positive < 0 {
		panic(fmt.Sprintf("invalid value: %d (must be >= 0)", positive))
	}
	fmt.Println("processed:", positive)
}

func main() {
	// Defer a function that can recover from a panic.
	defer func() {
		if r := recover(); r != nil {
			fmt.Fprintf(os.Stderr, "recovered from panic: %v\n", r)
			// Optionally re-panic if this is unrecoverable.
		}
	}()

	process(10)  // normal execution
	process(-5)  // triggers panic, then the deferred recover runs
	process(20)  // this line never executes

	fmt.Println("main continues after recovery")
}
```

**Output:**
```
processed: 10
recovered from panic: invalid value: -5 (must be >= 0)
main continues after recovery
```

Key points:

- `panic(any)` accepts any value, but by convention you panic with a string or an error.
- `recover()` returns the value passed to `panic` (or `nil` if no panic is in progress).
- You must call `recover` **directly** inside a deferred function; calling it from a function that is not deferred, or indirectly (e.g., via another function call from the deferred function), will not work.

A more realistic pattern is to use `recover` to close resources and then re‑panic after logging:

```go
func criticalOperation() (err error) {
	defer func() {
		if r := recover(); r != nil {
			// Convert panic to a typed error.
			err = fmt.Errorf("panic: %v", r)
		}
	}()
	// ... do work that might panic
	return nil
}
```

---

### 2. Under the Hood

When `panic` is called, the Go runtime performs the following steps:

1. **Stop normal execution** – no further code in the current function runs.
2. **Unwind the stack** – the runtime walks the call stack from the point of panic outward.
3. **Execute deferred functions** – at each stack frame, all `defer` functions are invoked in LIFO order. This is why `recover` can only work inside a `defer`: the panic unwinding mechanism runs deferrals before destroying the frame.
4. **If `recover` is called** inside a deferred function and it returns a non‑nil value, the runtime *stops the unwinding*. Control resumes after the `defer` that called `recover`. The function that panicked returns, and execution continues as normal.
5. **If no `recover` intercepts** the panic, the runtime prints the panic value, a stack trace, and calls `exit(2)`.

Notably, panics **do not cross goroutine boundaries**. A panic in one goroutine only unwinds that goroutine’s stack; other goroutines continue running unless the entire program exits.

The runtime maintains a per‑goroutine list of pending panics. When a panic occurs, the runtime pushes the panic value onto this list. As the stack unwinds, each deferred function can call `recover`, which pops the topmost panic from the list and returns it.

**Stack vs. heap considerations:** The panic value itself escapes to the heap because it must survive the stack unwinding. Recovering from a panic does *not* restore the stack to its previous state; it merely allows the goroutine to continue from the point after the `defer` that called `recover`.

---

### 3. Why This Design?

The Go team deliberately rejected exceptions (as found in Java, C++, and Python) for several philosophical reasons:

- **Errors are values** – Most failures are expected conditions (e.g., file not found, network timeout) and should be handled explicitly via `error` returns. Exceptions encourage “invisible” control flow, where a function can bail out unexpectedly.
- **Panic is for programmer bugs** – Index out of bounds, nil pointer dereference, or a failed assertion that violates internal invariants. In well‑written code, these *should never happen* at runtime. If they do, the safest response is often to crash the process (or the offending goroutine) rather than try to recover and continue in an unknown state.
- **Simplicity over magic** – Exceptions create hidden edges: every function call is a potential exit point. Go’s `if err != nil` makes control flow explicit. Panic is intended to be as rare and as loud as possible.
- **No checked exceptions** – Java’s checked exceptions force callers to either handle or declare them, leading to boilerplate and empty `catch` blocks. Go’s approach says: if a function might fail, return an error; if it’s truly catastrophic, panic.

Thus, panic/recover is a very thin layer—powerful enough to build per‑request recovery in servers, but not so convenient that developers use it for routine error handling.

---

### 4. Competing Approaches

| Language | Mechanism | Philosophy |
|----------|-----------|-------------|
| **Java** | Checked & unchecked exceptions + `try/catch/finally` | Failures are part of the method signature. The compiler enforces handling or propagation. `RuntimeException` for programmer errors. |
| **Python** | `try/except/else/finally` | Exceptions are the primary error handling mechanism, even for expected control flow (e.g., `StopIteration`). |
| **C++** | Exceptions + `try/catch` + RAII | Exceptions are for exceptional conditions, but resource cleanup uses destructors (similar to `defer`). |
| **Rust** | `panic!` macro + `Result<T, E>` | `Result` for recoverable errors (like Go’s `error`). `panic!` for bugs or unrecoverable failures. `catch_unwind` to recover from panics (with limits). |
| **Go** | `panic` + `recover` | Panic only for invariants and unrecoverable states. Recover is a “safety net” for cleaning up resources before crashing or for converting to an error at a boundary. |

**Key differences from Rust:**  
Rust’s panic *can* be caught with `catch_unwind`, but the language encourages you to use `Result` instead. Both languages share the idea that panics are for bugs, not for ordinary error handling. However, Rust’s ownership model prevents many panics (e.g., no nil pointer dereference) that Go must panic on.

**Key differences from Java:**  
Java programs often use exceptions for everything, from “file not found” to “out of memory”. Go separates these domains: ordinary failures → `error`, catastrophic failures → `panic`. Java’s `finally` block is analogous to `defer`, but `defer` is more lightweight and can be stacked arbitrarily.

---

### 5. Common Mistakes

Even experienced engineers trip over these panic/recover nuances.

#### Mistake 1: Recovering outside a deferred function

```go
func badRecover() {
    r := recover()
    fmt.Println("recovered:", r)  // always prints nil, does nothing
}
func main() {
    defer badRecover()   // badRecover calls recover - not direct
    panic("boom")
}
```

**Fix:** Recover must be called *directly* inside the deferred closure, not via a helper.

```go
defer func() {
    if r := recover(); r != nil {
        // handle
    }
}()
```

#### Mistake 2: Recovering a nil panic value incorrectly

`recover()` returns `nil` when no panic is active. A common anti‑pattern is:

```go
defer func() {
    recover()   // ignoring return value - does nothing when no panic
}()
```

**Fix:** Always check the returned value.

#### Mistake 3: Letting a goroutine panic kill the program

A panic in a goroutine that is not recovered will crash the entire process—even if other goroutines are healthy.

```go
go func() {
    // If this panics, main may not handle it.
    panic("goroutine died")
}()
```

**Fix:** Every goroutine that can panic should have its own `recover` at its top level (similar to starting a background thread with a catch‑all handler).

#### Mistake 4: Using panic for flow control

```go
func readFile(path string) string {
    data, err := os.ReadFile(path)
    if err != nil {
        panic(err)   // terrible: caller cannot handle missing file gracefully
    }
    return string(data)
}
```

**Fix:** Return an error. Panic is not a shortcut for early return.

#### Mistake 5: Recovering and continuing without fixing state

If your program panics because of a corrupted heap or invalid internal invariant, simply recovering and carrying on can lead to silent data corruption. Only recover when you can absolutely guarantee that the program’s state is still consistent (e.g., in an HTTP handler, recovering and returning a 500 error is safe because the handler’s state is request‑local).

---

### 6. Performance Considerations

- **Panic is expensive.** Throwing a panic incurs stack unwinding, defer execution, and printing overhead (if uncaught). In microbenchmarks, a panic can be 10–100x slower than returning an error.
- **Recover also has cost.** The runtime must examine the panic list and modify the control flow. Recovering is cheaper than letting the panic terminate the program, but still heavy.
- **Deferred functions run even without a panic.** The cost of a `defer` is small (roughly a few nanoseconds in Go 1.21+), but if you `defer` a function that calls `recover` on every hot path, you add unnecessary overhead. Only install recover‑capable defers at boundaries where panics are plausible (e.g., per‑HTTP request, not per‑loop iteration).
- **Escape analysis.** The value passed to `panic` escapes to the heap. If you panic with a large struct, it will allocate memory. Similarly, the deferred closure that calls `recover` may capture variables, preventing them from being stack‑allocated.

**Rule of thumb:** Do not use panic in performance‑sensitive inner loops. Reserve it for initialization, fatal errors, and boundary protection.

---

### 7. Best Practices (Idiomatic Go)

1. **Let panics crash the program in `main` or `init`** – If your program cannot start correctly (e.g., config file missing, required port in use), calling `panic` is acceptable because there’s no way to continue.

2. **In libraries, avoid panics entirely** – Library code should return errors. The only exception is if the library detects a programming mistake (e.g., a required `Register` function called twice). Even then, consider using a sync.Once with a panic to signal misuse.

3. **Use `MustXxx` functions sparingly** – A common pattern for functions that are guaranteed to succeed at runtime (e.g., `regexp.MustCompile`, `template.Must`). These panic on invalid input, shifting the burden to the programmer to provide valid arguments.

   ```go
   var re = regexp.MustCompile(`^[a-z]+$`)  // panics if regex is invalid
   ```

4. **Recover at goroutine boundaries** – Every goroutine you spawn (including those from `http.Handler` or worker pools) should have a top‑level `defer recover` that logs the panic and optionally shuts down the goroutine gracefully.

   ```go
   func worker(id int, jobs <-chan Job) {
       defer func() {
           if r := recover(); r != nil {
               log.Printf("worker %d panicked: %v", id, r)
               // Restart this worker or signal supervisor.
           }
       }()
       for job := range jobs {
           job.Process()
       }
   }
   ```

5. **Never recover a panic without logging** – Swallowing a panic turns a deterministic crash into silent, unpredictable behavior. Always log the panic value and stack trace (the runtime prints it automatically if you re‑panic, otherwise capture via `debug.Stack()`).

6. **Convert panics to errors only at API boundaries** – For example, when implementing a plugin system or calling third‑party code that might panic, use `recover` to convert the panic into an `error`.

7. **Avoid `recover()` in tests** – Use `testing`’s `t.Helper()` and `t.Fatal()` instead. If you must test that a function panics, use `assert.Panics` from testify or write a simple wrapper:

   ```go
   func assertPanics(t *testing.T, f func()) {
       defer func() {
           if r := recover(); r == nil {
               t.Error("expected panic, got none")
           }
       }()
       f()
   }
   ```

---

### 8. Examples

#### Example 1: HTTP middleware that recovers from panics

The standard `net/http` server does **not** recover from panics by default. A panic in a handler crashes the entire server process. Robust production servers wrap each request with a recovery middleware.

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"runtime/debug"
)

// RecoverMiddleware catches panics in the next handler, logs the stack,
// and returns a 500 Internal Server Error.
func RecoverMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if err := recover(); err != nil {
				// Log the panic and stack trace.
				log.Printf("panic serving %s: %v\n%s", r.URL.Path, err, debug.Stack())
				// Return a generic error to the client.
				http.Error(w, "Internal Server Error", http.StatusInternalServerError)
			}
		}()
		next.ServeHTTP(w, r)
	})
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/panic", func(w http.ResponseWriter, r *http.Request) {
		panic("simulated handler panic")
	})
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "OK")
	})

	// Wrap the entire mux with recovery.
	log.Fatal(http.ListenAndServe(":8080", RecoverMiddleware(mux)))
}
```

#### Example 2: Must pattern – compile template on startup

```go
package main

import (
	"html/template"
	"os"
)

// Must panics if the error is not nil. Only use during initialization.
func Must(t *template.Template, err error) *template.Template {
	if err != nil {
		panic(err)
	}
	return t
}

// Global template – panics if the file cannot be parsed.
var indexTemplate = Must(template.ParseFiles("index.html"))

func main() {
	// At this point, indexTemplate is guaranteed to be valid.
	if err := indexTemplate.Execute(os.Stdout, nil); err != nil {
		// This is a runtime error (e.g., write failure) – we return it normally.
		panic(err) // or handle gracefully
	}
}
```

#### Example 3: Graceful goroutine restart after panic

```go
package main

import (
	"log"
	"time"
)

type Worker struct {
	id  int
	stop chan struct{}
}

func (w *Worker) run() {
	defer func() {
		if r := recover(); r != nil {
			log.Printf("worker %d panicked: %v; restarting after 1s", w.id, r)
			time.Sleep(1 * time.Second)
			go w.run() // restart the worker
		}
	}()
	log.Printf("worker %d started", w.id)
	// Simulate work that sometimes panics.
	for {
		select {
		case <-w.stop:
			return
		default:
			// pretend to do work
			time.Sleep(500 * time.Millisecond)
			panic("simulated failure")
		}
	}
}

func main() {
	w := &Worker{id: 1, stop: make(chan struct{})}
	go w.run()
	time.Sleep(5 * time.Second)
	close(w.stop)
	log.Println("shutting down")
}
```

---

### 9. Summary & Exercises

**Summary**

- `panic` stops normal execution and unwinds the stack, running deferred functions.
- `recover`, called inside a `defer`, intercepts the panic and returns the panic value, allowing the goroutine to continue.
- Use panic only for **programmer errors** or **unrecoverable failures** (e.g., failed init).
- Never use panic for ordinary error handling – return `error` instead.
- Every long‑lived goroutine should have a recovery wrapper.
- Recovering without logging or without restoring invariants leads to silent failures.

**Key philosophy recap:** Go’s panic/recover is a surgical tool. It is not an exception system; it is a last‑resort safety net. Embrace explicit error handling, and let panics be loud and rare.

---

### Exercises

**Exercise 1: Goroutine supervisor**  
Write a function `SupervisedGo(fn func())` that starts `fn` in a new goroutine and automatically restarts it if it panics (up to a maximum of 3 restarts). After the third restart, the supervisor should log a fatal error and not restart again. *Hint: Use a loop with a defer/recover inside the goroutine.*

**Exercise 2: Safe assertion for critical invariants**  
Implement a package `assert` with a single exported function `Must(condition bool, msg string)`. If `condition` is false, `Must` should panic with `msg`. Then write a function that processes a slice of integers but panics if the slice length exceeds a predefined limit (use `Must`). Compare this approach to returning an error – when would you choose one over the other?

**Exercise 3: Convert panic to error in a read–eval–print loop (REPL)**  
Build a minimal REPL that reads a line, evaluates a Go expression (e.g., using `go/parser` and `go/types` – or simply a mock evaluator that panics on division by zero), and prints the result. If evaluating the expression panics, recover and print “evaluation error: <reason>”. The REPL should continue to the next input. *This mimics how Python’s REPL recovers from exceptions in user code.*
