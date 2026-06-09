# Chapter 16: Errors as Values

## 1. Basic Usage

In Go, errors are not language-level exceptions—they are values just like strings, integers, or structs. Any type implementing the `error` interface qualifies:

```go
type error interface {
    Error() string
}
```

**Creating and Returning Errors**

```go
import "errors"

// Sentinel error (fixed value)
var ErrTimeout = errors.New("operation timed out")

// Dynamic error with fmt.Errorf
func parsePort(portStr string) (int, error) {
    port, err := strconv.Atoi(portStr)
    if err != nil {
        return 0, fmt.Errorf("invalid port %q: %w", portStr, err)
    }
    if port < 1 || port > 65535 {
        return 0, fmt.Errorf("port %d out of range [1,65535]", port)
    }
    return port, nil
}

// Custom error type
type ValidationError struct {
    Field string
    Value any
    Msg   string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on %q (%v): %s", e.Field, e.Value, e.Msg)
}
```

**Error Wrapping (Go 1.13+)**

```go
func readConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        // %w wraps the original error
        return nil, fmt.Errorf("read config %s: %w", path, err)
    }
    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        return nil, fmt.Errorf("parse config %s: %w", path, err)
    }
    return &cfg, nil
}

// Checking wrapped errors
func handleRead() {
    cfg, err := readConfig("/app/config.json")
    if err != nil {
        if errors.Is(err, os.ErrNotExist) {
            log.Printf("config file missing, using defaults")
            return
        }
        log.Fatalf("fatal: %v", err)
    }
    // use cfg...
}
```

**Unwrapping and Type Assertion**

```go
err := someFunc()
var valErr *ValidationError
if errors.As(err, &valErr) {
    fmt.Printf("Field %s failed: %s\n", valErr.Field, valErr.Msg)
}
```

**Multiple Errors (Go 1.20+)**

```go
var errs []error
for _, f := range files {
    if err := process(f); err != nil {
        errs = append(errs, fmt.Errorf("file %s: %w", f.Name(), err))
    }
}
if len(errs) > 0 {
    return errors.Join(errs...)
}
```

## 2. Under the Hood

The `error` interface is deliberately minimal. Because errors are values, the compiler treats them like any other interface—no special runtime machinery for unwinding stacks or propagating exceptions.

**Memory Layout of an `error` Variable**

An error variable holds two words: a pointer to the type’s method table (itable) and a pointer to the underlying data. When you return `nil`, both words are zero. This is why returning a `nil` pointer of a concrete error type does **not** result in a `nil` interface:

```go
func badReturn() error {
    var e *ValidationError = nil
    return e // returns an interface with non-nil type pointer
}

func main() {
    if badReturn() == nil {
        fmt.Println("this won't print")
    } else {
        fmt.Printf("error is: %#v\n", badReturn()) // (*main.ValidationError)(nil)
    }
}
```

**Wrapping Implementation**

`fmt.Errorf` with `%w` creates an `*wrapError` struct:

```go
// From the runtime (simplified)
type wrapError struct {
    msg string
    err error
}

func (e *wrapError) Error() string { return e.msg }
func (e *wrapError) Unwrap() error { return e.err }
```

Each `%w` adds one layer. `errors.Is` walks the chain using `Unwrap` (which returns the inner error) until it finds a match. `errors.As` similarly traverses, performing type assertions at each layer.

**Sentinel Errors Are Pointers**

`errors.New` returns a pointer to a hidden, unexported struct. Two separate calls to `errors.New("same text")` produce **distinct** values. That’s why sentinel errors are declared as package-level variables—they ensure a single memory address.

**`errors.Join` Internals**

`errors.Join` aggregates multiple errors into a `joinError` type that implements `Unwrap() []error`. Calling `errors.Is` or `errors.As` on a joined error checks **all** errors in the slice recursively, making it behave like a logical OR.

## 3. Why This Design?

**Simplicity Over Hidden Control Flow**

Exceptions create at least three implicit control-flow paths: normal execution, exception thrown, and (in many languages) finally blocks. Go’s errors-as-values make every possible exit explicit. When you read:

```go
data, err := doSomething()
if err != nil {
    return err
}
```

you know exactly where the function can return and under what conditions. No scanning for `try/catch` blocks several frames up.

**Errors Are Not Exceptional**

The Go team observed that in networked, distributed systems—where Google operates—error conditions (timeouts, retries, missing data) are **normal**. Exceptions encourage developers to treat errors as “should never happen” and lead to fragile code that crashes or misbehaves when the inevitable occurs. Errors-as-values force you to design for failure.

**Preventing Swallowed Exceptions**

In Java, C++, or Python, it’s trivial to write `try { ... } catch (Exception e) {}` (or `pass` in Python). Silent failures become invisible. Go has no syntax to ignore an error implicitly; you must assign it to `_` or explicitly handle it. This makes ignoring errors a **visible** decision that reviewers can flag.

**Composable and Extensible**

Because `error` is an interface, you can attach any data or behavior. Want a stack trace? Implement `Error() string` that calls `runtime.Callers`. Want retryable vs fatal semantics? Add a `Temporary() bool` method (as the `net` package does). Want to add fields like HTTP status codes, request IDs, or retry-after durations? Embed them in a custom struct. Exceptions in most languages are rigid—you either extend a base class (single inheritance) or use complex machinery like exception filters.

## 4. Competing Approaches

| Language | Mechanism | Error Handling Philosophy | Trade-off |
|----------|-----------|--------------------------|------------|
| **Java** | Checked exceptions (must declare `throws`) + unchecked exceptions | Exceptions for exceptional cases; checked exceptions enforce handling at compile time | Verbose signatures; widely circumvented by throwing `RuntimeException`; no guarantee of handling due to empty catch blocks |
| **Python** | Exception objects with `try/except/finally` | "Easier to ask for forgiveness than permission" (EAFP) | Clean happy-path code; errors can propagate arbitrarily far; difficult to know what a function might raise without documentation |
| **Rust** | `Result<T, E>` enum with `?` operator | Errors as values with pattern matching; panic for unrecoverable | Most similar to Go but more expressive; `?` reduces verbosity; requires dealing with error type conversions (map_err, `anyhow`, `thiserror`) |
| **C++** | Exceptions with RAII | Exceptions for actual exceptional conditions (e.g., out of memory) | Exception safety requires careful design; binary size and runtime overhead; many codebases disable exceptions (-fno-exceptions) |
| **JavaScript** | `throw` / `try-catch` + Promise rejections | Exceptions everywhere, including async | Can throw anything (numbers, strings, objects); no type safety; async error handling requires `catch` or `.catch()` |

**Rust’s `Result` vs. Go’s `error`**

Rust’s `Result<T, E>` is a tagged union—the compiler knows whether you’ve checked it (if you don’t use the value, you get a warning). The `?` operator transforms `Result<T, E>` into `T` and early-returns the `Err(E)`. This reduces boilerplate while preserving explicitness. Go intentionally lacks this sugar because the team values visual noise: every `if err != nil` is a conscious checkpoint. Rust also requires you to map between error types (`map_err`), whereas Go’s `fmt.Errorf` with `%w` simply wraps without type conversion—flexible but less precise.

**Why Go Didn’t Choose Rust’s `?` Operator**

The Go designers argued that automatic propagation hides the error path. In Rust, `foo()?` can return from the function implicitly; in Go, you must write `if err != nil { return err }`. This aligns with Go’s philosophy of **explicit is better than implicit**, even at the cost of verbosity.

## 5. Common Mistakes

**1. Swallowing Errors with `_`**

```go
data, _ := ioutil.ReadFile("config.json") // Silent failure; data may be empty
```

**Fix**: Always check or explicitly document why you’re ignoring the error (e.g., `// ignoring error because file may not exist`).

**2. Using `==` on Wrapped Errors**

```go
err := readConfig()
if err == os.ErrNotExist { // Never true if err is wrapped
    // ...
}
```

**Fix**: Use `errors.Is(err, os.ErrNotExist)`.

**3. Type Asserting Without Checking**

```go
if err.(*ValidationError).Field == "email" { // Panics if err is nil or wrong type
```

**Fix**: Use `errors.As` or a type switch:

```go
var valErr *ValidationError
if errors.As(err, &valErr) {
    // use valErr.Field
}
```

**4. Returning `nil` Pointer of Error Type**

```go
func bad() error {
    var e *MyError // e is nil
    return e       // returns non-nil interface
}
```

**Fix**: `return nil`.

**5. Not Wrapping Errors with Context**

```go
if err := db.Query(); err != nil {
    return err // loses where this happened
}
```

**Fix**: `return fmt.Errorf("query user %d: %w", userID, err)`.

**6. Excessive Wrapping**

```go
return fmt.Errorf("failed: %w", fmt.Errorf("inner: %w", err)) // Two wraps when one suffices
```

**Fix**: Add only the context that helps debugging.

**7. Ignoring `errors.Join` Order**

`errors.Join` returns an error that `Unwrap() []error`. If any contained error matches `errors.Is`, the whole join matches. This is a logical OR. However, **no stack trace or cause chain** is preserved across the join. For detailed debugging, you may need a custom multi‑error type with structured fields.

## 6. Performance Considerations

**Allocation Cost**

Every call to `errors.New`, `fmt.Errorf`, or custom error constructor allocates memory (unless the error is a package-level singleton). In hot paths—e.g., validating thousands of requests per second—this can create GC pressure.

**Mitigations**:
- Use sentinel errors (`var ErrNotFound = errors.New("not found")`) for static errors.
- Avoid wrapping inside tight loops; instead, return sentinel errors and add context at the outermost layer.
- Use `sync.Pool` for frequently created custom error types.

**`errors.Is` and `errors.As` Complexity**

Both functions walk the error chain using `Unwrap`. The chain length equals the number of `%w` wraps. For deeply wrapped errors (10+ levels), repeated `Is` checks become O(n * depth). In practice, depth rarely exceeds 3–5 unless you’re wrapping inside recursive calls.

**Stack Traces Are Opt-In**

Go’s standard `error` does not capture stack traces. This makes `errors.New` extremely cheap—just a string allocation and a pointer. Third-party packages like `github.com/pkg/errors` capture the stack (using `runtime.Callers`), which costs roughly 1–2 microseconds and allocates additional frames. Use stack traces only in long-running servers where debugging occasional errors is worth the overhead.

**Exception Performance Comparison**

In Java/C++, throwing an exception involves:
- Walking the call stack to find a catch block
- Running destructors (C++) or finally blocks
- Allocating an exception object (often on the heap)

This is expensive—often thousands of cycles. Go’s error-as-value returns a single word (or two) and a conditional branch, typically under 10 cycles if the error path is cold. For **expected** errors (validation failures, missing cache entries), Go is dramatically faster.

**GC Impact of Wrapping**

Each `fmt.Errorf("%w")` adds a small `wrapError` struct (two pointers). Wrapping a 10‑layer error chain creates 10 distinct heap objects. If these errors are created only on failure paths, the GC pressure is negligible. However, consider this anti-pattern:

```go
for _, item := range hugeSlice {
    if err := validate(item); err != nil {
        return fmt.Errorf("validation failed: %w", err) // allocates on every failure
    }
}
```

If failures are frequent, allocate once: `return &ValidationError{...}`.

## 7. Best Practices

**Sentinel Errors for Package-Level Fixed Conditions**

```go
package storage

var ErrNotFound = errors.New("storage: key not found")
var ErrAlreadyExists = errors.New("storage: key already exists")
```

Use `errors.Is` for checking.

**Custom Error Types for Structured Data**

```go
type APIError struct {
    StatusCode int
    Message    string
    RequestID  string
}

func (e *APIError) Error() string {
    return fmt.Sprintf("API error %d: %s (req=%s)", e.StatusCode, e.Message, e.RequestID)
}

// Optional: implement Unwrap if you want to chain
func (e *APIError) Unwrap() error { return errors.New(e.Message) }
```

**Wrap Only at API Boundaries**

Internal library functions should return sentinel or typed errors. At the service boundary (HTTP handler, CLI command, RPC endpoint), wrap with context:

```go
func (s *Service) GetUser(ctx context.Context, id string) (*User, error) {
    u, err := s.db.UserByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("get user %s: %w", id, err)
    }
    return u, nil
}
```

**Use `errors.Is` and `errors.As` Exclusively**

Never write `err == someErr` unless `someErr` is a guaranteed non‑wrapped sentinel and you own the entire error chain. Even then, `errors.Is` is clearer and future-proof.

**Guard Clauses Pattern**

```go
func ProcessOrder(order *Order) error {
    if order == nil {
        return errors.New("order is nil")
    }
    if err := validate(order); err != nil {
        return fmt.Errorf("validation: %w", err)
    }
    // happy path continues
    return nil
}
```

**Error Messages: Lowercase, No Trailing Punctuation**

```go
// Good
return fmt.Errorf("failed to parse time: %w", err)

// Avoid
return fmt.Errorf("Failed to parse time: %w.", err)
```

This convention makes wrapping (`fmt.Errorf("context: %w", err)`) produce grammatically consistent messages.

**Document Error Behavior**

For public functions, explicitly state:
- Which sentinel errors they return
- Whether they wrap errors (and thus respond to `errors.Is` for lower-level sentinels)

```go
// GetUser returns the user with the given ID.
// It returns ErrNotFound if the user does not exist.
// Any other error is wrapped with context.
```

**Avoid Panics in Libraries**

Libraries should return errors for recoverable conditions. Panics are reserved for programming bugs (e.g., uninitialized maps, out-of-bounds slice access where preconditions are documented). `panic("validation failed")` is never appropriate.

## 8. Examples

**Example 1: File Parser with Custom Error Types**

```go
package main

import (
    "encoding/json"
    "errors"
    "fmt"
    "os"
)

type ParseError struct {
    File   string
    Line   int
    Err    error
}

func (e *ParseError) Error() string {
    return fmt.Sprintf("parse error in %s at line %d: %v", e.File, e.Line, e.Err)
}

func (e *ParseError) Unwrap() error { return e.Err }

type Config struct {
    Timeout int `json:"timeout"`
}

func loadConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("read %s: %w", path, err)
    }

    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        var syntax *json.SyntaxError
        if errors.As(err, &syntax) {
            // Approximate line number from byte offset
            line := 1 + syntax.Offset
            return nil, &ParseError{File: path, Line: int(line), Err: err}
        }
        return nil, fmt.Errorf("parse %s: %w", path, err)
    }

    if cfg.Timeout <= 0 {
        return nil, fmt.Errorf("invalid timeout %d in %s", cfg.Timeout, path)
    }
    return &cfg, nil
}

func main() {
    cfg, err := loadConfig("config.json")
    if err != nil {
        var parseErr *ParseError
        if errors.As(err, &parseErr) {
            fmt.Printf("Fixing line %d\n", parseErr.Line)
        } else if errors.Is(err, os.ErrNotExist) {
            fmt.Println("Creating default config")
            cfg = &Config{Timeout: 30}
        } else {
            fmt.Fprintf(os.Stderr, "fatal: %v\n", err)
            os.Exit(1)
        }
    }
    fmt.Printf("Config loaded: %+v\n", cfg)
}
```

**Example 2: Multi-Error Aggregation**

```go
package main

import (
    "errors"
    "fmt"
    "strings"
)

func validateUser(name, email string) error {
    var errs []error

    if strings.TrimSpace(name) == "" {
        errs = append(errs, errors.New("name cannot be empty"))
    }

    if !strings.Contains(email, "@") {
        errs = append(errs, fmt.Errorf("email %q missing @", email))
    }

    if len(errs) == 0 {
        return nil
    }
    return errors.Join(errs...)
}

func main() {
    err := validateUser("", "alice")
    if err != nil {
        // errors.Join produces a string with newlines
        fmt.Printf("Validation failed:\n%v\n", err)
        // Use errors.Is? Not for individual sentinels. Check each manually:
        if errors.Is(err, errors.New("name cannot be empty")) { // Does NOT work - new sentinel each time
            fmt.Println("Name error detected (but this won't fire)")
        }
        // Better: use custom multi-error type or inspect string.
    }
}
```

**Example 3: Retry with Wrapped Transient Errors**

```go
package main

import (
    "errors"
    "fmt"
    "net"
    "time"
)

type TransientError struct {
    Err error
}

func (e *TransientError) Error() string { return fmt.Sprintf("transient: %v", e.Err) }
func (e *TransientError) Unwrap() error { return e.Err }
func (e *TransientError) Temporary() bool { return true }

func isTransient(err error) bool {
    var terr interface{ Temporary() bool }
    if errors.As(err, &terr) {
        return terr.Temporary()
    }
    // Network errors often have Temporary()
    var netErr net.Error
    if errors.As(err, &netErr) {
        return netErr.Temporary()
    }
    return false
}

func doRequest() error {
    // Simulate a transient failure
    return &TransientError{Err: net.ErrClosed}
}

func retry(attempts int, fn func() error) error {
    var err error
    for i := 0; i < attempts; i++ {
        err = fn()
        if err == nil {
            return nil
        }
        if !isTransient(err) {
            return fmt.Errorf("non‑transient failure: %w", err)
        }
        time.Sleep(time.Duration(1<<i) * 100 * time.Millisecond)
    }
    return fmt.Errorf("retry exhausted: %w", err)
}

func main() {
    if err := retry(3, doRequest); err != nil {
        fmt.Println("Failed:", err)
    } else {
        fmt.Println("Success")
    }
}
```

## 9. Summary & Exercises

### Summary

- **Errors are values** implementing the one‑method `error` interface.
- **Explicit handling** via `if err != nil` makes every failure point visible.
- **Wrapping** with `fmt.Errorf("%w", err)` adds context without losing the original error chain.
- **`errors.Is`** checks for a specific error value anywhere in the chain (value equality).
- **`errors.As`** extracts the first error of a specific type from the chain.
- **Sentinel errors** (package variables) are for fixed, unchangeable error conditions.
- **Custom error types** carry structured data and can implement `Unwrap` or `Temporary() bool`.
- **`errors.Join`** aggregates multiple independent errors.
- **Performance** is excellent for expected errors; avoid allocation in hot paths by reusing sentinels.
- **Best practices**: always wrap at boundaries, document error behavior, never use `panic` for recoverable errors, and prefer `errors.Is` over `==`.

### Exercises

**Exercise 1: Aggregated File Processor**

Build a function `ProcessFiles(files []string) error` that attempts to process each file with a provided `processFunc(filename string) error`. Use `errors.Join` to collect all errors. Modify the function to return a custom `MultiError` type that also records the count of failed files and the first failure time. Ensure that `Error()` prints a useful summary and that `Unwrap() []error` returns the individual errors.

**Exercise 2: Retry with Distinguishable Errors**

Implement a `http.Get` wrapper that retries on network timeouts and 5xx status codes, but fails immediately on 4xx client errors. Define a `HTTPError` struct containing `StatusCode` and `URL`. Implement a method `Retryable() bool`. Use `errors.As` inside your retry loop to decide whether to retry. Add jitter to the backoff and a maximum retry duration.

**Exercise 3: Refactor Java Exception Code**

Given the following Java-style pseudo‑code (using exceptions), rewrite it in idiomatic Go with errors-as-values. Preserve the same behavior: reading a configuration file, parsing JSON, validating fields, and connecting to a database. Do not panic; return meaningful errors at each step that can be checked by the caller to decide between fatal exit, retry, or fallback defaults.

```java
// Java pseudo-code
public Config loadConfig(String path) {
    try {
        String json = Files.readString(Path.of(path));
        Config cfg = objectMapper.readValue(json, Config.class);
        if (cfg.getPort() < 1024) {
            throw new ConfigException("port too low");
        }
        return cfg;
    } catch (IOException e) {
        throw new RuntimeException("cannot read file", e);
    } catch (JsonParseException e) {
        throw new ConfigException("invalid JSON", e);
    }
}
```

Implement the Go version and demonstrate how the caller distinguishes between “file not found” (use default config), “invalid JSON” (log and exit), and “port too low” (exit with error message).
