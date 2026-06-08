## Chapter 16. Errors as Values

The seasoned engineer arriving from ecosystems built on exceptions will find Go’s error handling jarring—not because it lacks sophistication, but because it wears its simplicity on its sleeve. Go treats errors as ordinary values, no different from an `int` or a `string`. This choice is not a compromise; it is a deliberate philosophical stance that says error handling is part of your program’s normal flow, not a separate, magic-ridden side channel. In this chapter, we unpack how to wield the `error` interface effectively, why wrapping, unwrapping, and the guard clause pattern produce resilient systems, and how Go’s explicit style eliminates entire categories of bugs that exceptions invite.

---

### 1. Basic Usage

The `error` interface is the single most important contract in the language:

```go
type error interface {
    Error() string
}
```

Any type with an `Error() string` method satisfies it. At its simplest, you create errors with `errors.New` or `fmt.Errorf`, return them from functions, and check them with `if err != nil`. There is no magic—just values.

```go
package main

import (
    "errors"
    "fmt"
)

var ErrNotFound = errors.New("item not found")

func findUser(id int) (string, error) {
    if id <= 0 {
        return "", fmt.Errorf("findUser: invalid id %d: %w", id, ErrNotFound)
    }
    return "alice", nil
}

func main() {
    name, err := findUser(0)
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            fmt.Println("not found, handle gracefully")
        } else {
            fmt.Println("unexpected error:", err)
        }
        return
    }
    fmt.Println("user:", name)
}
```

Three mechanisms are at play:
- **Creating errors:** `errors.New` for static sentinels, `fmt.Errorf` with the `%w` verb for wrapping context around an existing error.
- **Checking sentinels:** `errors.Is` traverses the error chain to see if any wrapped error matches the sentinel.
- **Extracting types:** `errors.As` tests whether an error (or any error it wraps) can be assigned to a specific type, filling a target variable.

From Go 1.20, `errors.Join` lets you combine multiple simultaneous errors into one:

```go
err1 := errors.New("connection refused")
err2 := errors.New("timeout")
combined := errors.Join(err1, err2)
// combined.Error() == "connection refused\ntimeout"
fmt.Println(errors.Is(combined, err1)) // true
```

This suite of standard functions forms the complete vocabulary for error handling in Go—no framework required.

---

### 2. Under the Hood

When you write `return fmt.Errorf("findUser: %w", err)`, the compiler does not allocate a stack trace, a thread-local exception object, or any hidden bookkeeping. The `fmt.Errorf` with `%w` creates a concrete type (from the `errors` package’s unexported `wrapError`) that holds two fields: a message string and the wrapped error. That struct implements the `Error() string` method by concatenating the message and the underlying error’s message, and it exposes an `Unwrap() error` method. This is composition in its purest form—wrapping builds a singly linked list of errors.

- **`errors.Is`** walks this list using `Unwrap() error` (or the `Is(error) bool` method if the error implements a custom `Is`). It stops when it finds a match.
- **`errors.As`** similarly unwraps and checks each error for assignability to the target type, using either `As(any) bool` if implemented, or standard reflection.
- **`errors.Join`** returns an error that implements `Unwrap() []error`, which `Is` and `As` traverse breadth-first, ensuring all joined errors are consulted.

There is no runtime overhead for the `nil` error path—returning `nil` is just a zero-value interface assignment. When an error is not nil, the interface value carries a concrete type pointer (e.g., to the `wrapError`) and the method dispatch for `Error()` happens through the interface’s itable. The error string is typically constructed lazily or cached, but `Error()` can be called multiple times, so implementations must not contain expensive logic there.

Crucially, the standard library’s errors do **not** capture a stack trace. That means constructing an error is as cheap as allocating a small struct and a string. No goroutine stack walking, no symbolication. This is a deliberate trade‑off: you lose the automatic “where did it happen?” but gain predictable performance, especially in the hot path where you might construct errors but immediately handle them.

---

### 3. Why This Design?

Go’s rejection of exceptions is a cornerstone of its philosophy. Exceptions introduce a hidden, non‑local transfer of control. You cannot look at a function call and know where execution will resume when an error occurs; a `try` block somewhere up the call stack may catch it, or it may crash the program. This model encourages developers to forget about error handling until it’s too late. Languages compensate with resource management constructs like `finally`, `using`, or RAII, but these add cognitive load and, in large systems, lead to subtle leaks when resources are not correctly released.

By making errors values, Go brings error handling into the light. Every function’s signature declares that it may fail, and the caller is forced to confront that fact immediately. The guard clause pattern—`if err != nil { return err }`—is not boilerplate; it is a visible, explicit statement that the program has thought about what to do when things go wrong. This explicitness aligns with the first pillar: **simplicity over complexity**. There is no hidden magic, no stack unwinding semantics to study, just the same control flow you use for any other condition.

The design also embraces **composition**. Wrapping errors with `%w` lets you layer context without losing the original failure. A file read failure can become “loading config: open /etc/app.yaml: permission denied”—the original error (`permission denied`) remains programmatically inspectable through `errors.Is` and `errors.As`. This is the error equivalent of embedding structs: you build richer meaning by combining simple parts.

Rob Pike’s essay *Errors are values* drove this point home: you can pass errors around, store them in variables, even use them as flow control. For example, you can buffer errors and process them in bulk rather than returning immediately. Go’s concurrency pillar—“share memory by communicating”—finds a parallel here: don’t communicate by panicking; communicate by returning errors through channels just like any other data.

---

### 4. Competing Approaches

- **Python / JavaScript / C# / Java (exceptions):** These languages treat exceptional paths as a secondary control channel. Try/catch blocks separate the “happy path” from error handling. The runtime captures a stack trace, which aids debugging but adds cost. Exception safety requires careful use of `finally` or `using` to release resources. In contrast, Go requires you to handle the error exactly where it occurs, or explicitly propagate it upward. The upside: no hidden control flow, no surprise propagation, and resource cleanup is done with `defer`, which executes deterministically regardless of error.

- **Java checked exceptions:** Java attempted to force explicit handling but the mechanism was so verbose that developers routinely wrapped exceptions in unchecked `RuntimeException` just to avoid the ceremony. Go’s approach is similar in spirit—you must acknowledge errors—but without a separate language construct. The `error` value is just a return value, no new syntax needed.

- **C++ (exceptions + `std::expected`):** Modern C++ offers `std::expected<T,E>` as a value-based alternative, much like Rust’s `Result`. Go’s error handling is less generic: you don’t have a parametric sum type; instead, you return `(T, error)`. This is simpler but sacrifices monadic composition (no `.and_then` chaining). C++ exceptions still dominate legacy codebases, making a hybrid style common. Go’s uniformity means one style across the ecosystem.

- **Rust (`Result` and `?`):** Rust’s `Result<T,E>` is a powerful sum type that forces exhaustive handling at compile time, with the `?` operator for early returns. It’s arguably more expressive and type‑safe than Go’s approach. However, Go prioritizes visibility and simplicity over maximum type safety. The `if err != nil` line is unmistakable; `?` hides the propagation inside a single character. Both eliminate the exceptional hidden flow, but Go’s way keeps the error path explicit and tangible.

- **C (error codes):** Go is closest to classic C error handling, where functions return an error code or null and you check immediately. The difference is that Go’s `error` interface is type‑safe and composable. C’s `errno` is global, fragile, and non‑extensible. Go’s approach modernizes the pattern without abandoning its directness.

In summary, while other languages provide richer type‑level machinery for error handling, Go chooses a minimal, explicit, and highly readable convention that mirrors the straightforwardness of the language itself.

---

### 5. Common Mistakes

1. **Ignoring errors without intent.** Writing `_ = doWork()` is a red flag. If you truly don’t care, at least add a comment explaining why, or better, log the error. Silently discarding errors leads to mysterious failures.

2. **Losing context by returning bare errors.** Returning `err` directly from a function without adding any context makes debugging a nightmare. A stack of “permission denied” with no clue where it happened is useless. Always wrap: `fmt.Errorf("opening %s: %w", filename, err)`.

3. **Comparing wrapped errors with `==`.** Once you wrap an error with `%w`, the original sentinel is nested. `err == io.EOF` will fail if the error is wrapped. Use `errors.Is(err, io.EOF)` to search the chain.

4. **Over‑wrapping.** Adding a new prefix at every function boundary results in a giant, redundant message like “loadConfig: openConfig: readFile: permission denied”. Wrap only at boundaries where you have meaningful new context. Inside a package, you might return bare errors; the package API should wrap them to form a clear public message.

5. **Using `errors.As` incorrectly.** The second argument must be a pointer to a type implementing error. `var target *MyError; errors.As(err, target)` will not work; you need `errors.As(err, &target)`. Also, `errors.As` does not panic if target is a non‑pointer, it returns false, which can silently miss the match.

6. **Mixing `panic` and error handling.** Using `panic` for validation or non‑fatal conditions forces callers to `recover`—a poor man’s exception system. Keep `panic` for truly unrecoverable programmer errors, and use error values for everything else (see Chapter 17).

7. **Checking error strings.** Code that does `if strings.Contains(err.Error(), "timeout")` is fragile. Sentinel errors or custom types exist exactly to avoid string parsing. Invest a few minutes in defining a proper error type.

8. **Custom error types that break unwrapping.** If your custom error type wraps another error and you provide an `Error()` method but forget to implement `Unwrap()`, then `errors.Is` and `errors.As` cannot see through your type. Always implement `Unwrap()` (or `Unwrap() []error` for multi‑errors) when wrapping.

---

### 6. Performance Considerations

- **Error construction cost.** In the no‑error case, you return `nil`, which is zero‑cost. When an error occurs, `errors.New` allocates a small string and a struct. `fmt.Errorf` with `%w` incurs a `wrapError` allocation plus string formatting. In hot paths (e.g., parsing millions of lines), pre‑allocate sentinel errors and return them directly without wrapping each time. For example, a scanner can have a package‑level `var ErrInvalidFormat = errors.New("invalid format")` and return it; the caller can later wrap if needed.

- **Traversal of error chains.** `errors.Is` and `errors.As` follow the `Unwrap` linked list. Depth is usually shallow (<10), so the cost is negligible. If you build a deeply nested chain with many wraps, the linear walk can add up. Keep chains reasonable by not wrapping unnecessarily.

- **String construction.** The `Error()` method is called often by logging frameworks, so it should be fast. Avoid building huge messages with expensive operations inside `Error()`—do the work in the constructor and store the result. Standard library errors pre‑compute the string once.

- **Stack traces.** Third‑party packages like `github.com/pkg/errors` add stack traces to errors, which is handy for debugging but devastating for performance if errors are frequent. If you need stack traces, consider enabling them only in debug builds or using conditional logging. The standard library’s lightweight errors are your default.

- **Interface dispatch.** Comparing a non‑nil error to `nil` is a simple pointer check. Calling `err.Error()` goes through an interface dispatch, but this is just an indirect function call—minimal overhead.

In short, Go’s error handling is designed to be fast enough for the vast majority of systems code, and it becomes noticeable only when you are generating errors at a very high rate. In those cases, the usual remedy is to handle the error condition without creating an error value at all (e.g., check a validity flag before the operation) or to use a single, shared error value.

---

### 7. Best Practices

- **Always handle errors.** An unhandled error is a latent bug. Even if you only log it, make the decision explicit.

- **Use the guard clause.** Instead of nested `if-else`, return early on error. This keeps the happy path unindented and easy to read.

  ```go
  data, err := readFile(path)
  if err != nil {
      return fmt.Errorf("loading data: %w", err)
  }
  // continue with data
  ```

- **Wrap at API boundaries with `%w`.** When a public function returns an error from an internal call, add context that explains *what* failed: `fmt.Errorf("userService.Fetch(%d): %w", id, err)`. This builds a traceable chain without revealing internal implementation details.

- **Prefer sentinel errors for discrete failure modes.** `var ErrNotFound = errors.New("not found")` lets callers check the condition with `errors.Is` without depending on a specific string or type. Combine with wrapping to add context.

- **Define custom error types for structured data.** When you need to attach metadata (e.g., an HTTP status code, a retry‑after duration), create an exported type that implements `error` and, optionally, `Unwrap` if it wraps an underlying error. Ensure it works with `errors.As`.

- **Use `errors.Join` to collect multiple errors.** When you are running several independent operations (e.g., validating multiple inputs), collect all errors rather than stopping at the first one. This gives the caller a complete picture.

- **Let logging be a separate concern.** In library code, return errors without logging them. The application code that consumes the library decides whether to log, increment metrics, or handle silently. If you must log inside a library for debugging, use a structured logger passed as a dependency, but never assume the error should be silenced.

- **Design errors for the caller.** Think about what the caller will test for. Will they need to distinguish between temporary and permanent failures? Define sentinels or types accordingly. Avoid exposing raw error messages as part of your API contract; they are for human consumption and may change.

- **Embrace `errors.Is` and `errors.As` as the only tools for inspection.** Never compare error strings or cast directly without `As`. This decouples your code from implementation details and future‑proofs it when wrapping is added.

---

### 8. Examples

**Example 1: The guard clause with sentinel errors**

```go
package config

import (
    "errors"
    "fmt"
    "os"
)

var ErrConfigNotFound = errors.New("config not found")

func Load(path string) ([]byte, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        if errors.Is(err, os.ErrNotExist) {
            return nil, fmt.Errorf("%s: %w", path, ErrConfigNotFound)
        }
        return nil, fmt.Errorf("reading %s: %w", path, err)
    }
    return data, nil
}
```

Callers can then check: `if errors.Is(err, config.ErrConfigNotFound) { /* use defaults */ }`.

**Example 2: Custom error type and `errors.As`**

```go
type HTTPError struct {
    StatusCode int
    Message    string
    Err        error
}

func (e *HTTPError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("HTTP %d: %s: %v", e.StatusCode, e.Message, e.Err)
    }
    return fmt.Sprintf("HTTP %d: %s", e.StatusCode, e.Message)
}

func (e *HTTPError) Unwrap() error {
    return e.Err
}

func fetch(url string) error {
    // ... making HTTP request
    return &HTTPError{
        StatusCode: 404,
        Message:    "not found",
        Err:        fmt.Errorf("GET %s", url),
    }
}

func main() {
    err := fetch("https://api.example.com")
    var httpErr *HTTPError
    if errors.As(err, &httpErr) {
        fmt.Printf("status: %d, retryable: %v\n", httpErr.StatusCode, httpErr.StatusCode < 500)
    }
}
```

**Example 3: Combining errors with `errors.Join`**

```go
func validateForm(name, email string) error {
    var errs []error
    if name == "" {
        errs = append(errs, errors.New("name is required"))
    }
    if email == "" {
        errs = append(errs, errors.New("email is required"))
    }
    return errors.Join(errs...)
}
```

The caller receives a single error that aggregates all validations, and can inspect each with `errors.Is`.

---

### 9. Summary & Exercises

Errors as values force a discipline that scales. Instead of a hidden, non‑local control flow, you get a linear, predictable chain where every decision point is visible. Wrapping and the `errors` API let you build rich diagnostic information without sacrificing programmatic inspection. The guard clause style keeps your code flat and navigable, and sentinel/typed errors create a clean, testable contract between packages.

**Exercises:**

1. **Robust File Processor:** Write a function that reads a JSON configuration file and unmarshals it into a struct. The function must handle file not found, permission errors, and invalid JSON gracefully. Create sentinel errors for missing file and malformed JSON. Wrap errors with context so that a caller at the top level can log a descriptive message. Use `errors.Is` in the caller to decide whether to exit with code 1 (config missing) or code 2 (invalid).

2. **Retry with Error Classification:** Implement a network client that retries a request up to three times with exponential backoff. Use a custom error type `RetryableError` that wraps the underlying error and indicates whether the error is transient (e.g., timeout) or permanent (e.g., 400 Bad Request). Inside the retry loop, use `errors.As` to extract the custom type and decide whether to continue retrying. Write table‑driven tests that inject different error conditions and verify the retry count.

3. **Hierarchical Error Strategy for a Database Layer:** Design a package `db` that provides a `Get(key string) ([]byte, error)` method. Define two sentinel errors: `ErrNotFound` and `ErrUnavailable`. The implementation wraps standard library errors from a real driver (like `sqlite3`). Now build a caching layer on top that intercepts errors: if it sees `db.ErrNotFound`, it returns the cached value; if it sees `db.ErrUnavailable`, it logs and returns stale cache. Use `errors.Is` to make decisions. Demonstrate how the caching layer remains decoupled from the specific driver errors.
