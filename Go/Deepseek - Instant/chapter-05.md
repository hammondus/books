# Chapter 5: Functions

Functions are the primary building block of Go programs. Unlike class-based languages where behavior is attached to receivers by default, Go treats functions as first-class citizens that can stand alone, be assigned to variables, and capture scope. This chapter explores not just *how* to write functions, but why Go’s function design shapes the language’s entire approach to error handling, abstraction, and composability.

---

## 1. Basic Usage

### Function Declarations and Signatures

A function declaration consists of the `func` keyword, a name, optional parameters, optional return types, and a body.

```go
// Basic function with parameters and a single return
func add(x int, y int) int {
    return x + y
}

// Parameters of the same type can be collapsed
func multiply(x, y int) int {
    return x * y
}
```

### Multiple Return Values

Go’s most visible function feature is native support for multiple return values. The standard library uses this extensively to return both a result and an error.

```go
import "errors"

// Divide returns quotient and remainder, or an error if divisor is zero
func divide(dividend, divisor int) (int, int, error) {
    if divisor == 0 {
        return 0, 0, errors.New("division by zero")
    }
    quotient := dividend / divisor
    remainder := dividend % divisor
    return quotient, remainder, nil
}

func main() {
    q, r, err := divide(10, 3)
    if err != nil {
        panic(err) // Real code would handle gracefully
    }
    println(q, r) // 3 1
}
```

### Named Return Values

Return values can be named, which serves two purposes: documentation and implicit returns. Named returns act as variables initialized to their zero values.

```go
// Named returns document what each return value means
func divideNamed(dividend, divisor int) (quotient, remainder int, err error) {
    if divisor == 0 {
        err = errors.New("division by zero")
        return // naked return, returns current values of quotient, remainder, err
    }
    quotient = dividend / divisor
    remainder = dividend % divisor
    return // returns quotient, remainder, err (nil)
}
```

### Variadic Functions

Functions that accept a variable number of arguments use the `...T` syntax. Inside the function, the variadic parameter behaves as a slice.

```go
// Sum accepts any number of integers
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

func main() {
    println(sum(1, 2, 3, 4)) // 10
    
    // Expand a slice into variadic arguments
    numbers := []int{5, 6, 7}
    println(sum(numbers...)) // 18
}
```

### Closures (Anonymous Functions)

Functions can be defined inline and capture variables from the surrounding scope.

```go
// Adder returns a closure that captures the base value
func adder(base int) func(int) int {
    return func(delta int) int {
        return base + delta
    }
}

func main() {
    addFive := adder(5)
    println(addFive(3)) // 8
    println(addFive(10)) // 15
}
```

### Defer

Deferred functions execute after the surrounding function returns, regardless of whether the return is normal or caused by a panic. This is Go’s idiomatic cleanup mechanism.

```go
func readFile(filename string) error {
    f, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer f.Close() // Guaranteed to run, even if function panics
    
    // Process file...
    return nil
}
```

Defer arguments are evaluated immediately, but the call is postponed. This matters when deferring closures that reference changing variables.

```go
func deferPitfall() {
    x := 1
    defer fmt.Println(x) // Prints 1, not 2
    x = 2
}
```

---

## 2. Under the Hood

### Stack Frames and Function Calls

Go uses a **contiguous stack** that starts small (typically 2KB per goroutine) and grows as needed. Each function call pushes a stack frame containing:
- Local variables (including named return values)
- Parameter values
- Return address

Unlike C’s fixed-size stack, Go’s stack can be dynamically resized by the runtime. When the stack runs out of space, the runtime allocates a larger contiguous block, copies the existing stack, and adjusts pointers—a process transparent to the program.

### Multiple Return Value Implementation

Go compiles multiple returns into a **single contiguous return area** in the caller’s stack frame. When you write:

```go
func f() (int, error)
```

The compiler transforms this internally to something like:

```c
// Pseudo-C representation
void f(uintptr* ret_addr, int* ret_int, error* ret_err)
```

The caller allocates space for both return values before making the call. This is more efficient than heap-allocating a tuple or using exceptions, which require unwinding.

### Named Returns: Zero-Initialized Variables

Named return values are simply local variables allocated in the stack frame, **initialized to their zero values** before any user code runs. The `return` statement without arguments becomes a copy of those variables into the caller’s return area.

```go
func zeroTest() (x int, err error) {
    // x is 0, err is nil here
    if condition() {
        return // returns 0, nil
    }
    x = 42
    return // returns 42, nil
}
```

The performance cost is identical to unnamed returns—named returns do not add heap allocations.

### Closure Representation

A closure in Go is represented as a **function pointer plus an environment pointer** (or multiple pointers). When you write:

```go
func counter() func() int {
    count := 0
    return func() int {
        count++
        return count
    }
}
```

The compiler allocates the captured variable `count` on the **heap**, not the stack. The returned closure is a struct containing:
- A pointer to the function code
- A pointer to the captured variable

If the closure captures multiple variables, they are bundled into a single heap-allocated context struct. This is why closures in tight loops can cause GC pressure—capturing forces allocation.

### Variadic Parameter Sugar

The variadic syntax `...T` desugars to a slice parameter of type `[]T`. The caller, when providing individual arguments, causes the compiler to:
1. Create a hidden temporary slice on the stack (or heap if it escapes)
2. Populate it with the arguments
3. Pass a pointer to this slice

When you pass an existing slice with `slice...`, no new allocation occurs—the same slice header is passed.

### Defer Implementation

Each `defer` statement pushes a **defer record** onto a linked list attached to the goroutine. The record contains:
- The function pointer
- The arguments (already evaluated)
- A pointer to the next defer

When the function returns, the runtime executes these records in **LIFO order** (last deferred, first executed). Defer has a non-trivial cost (~50-100ns on modern hardware) because it involves heap allocation of the defer record in some implementations. Go 1.14+ optimized most defers to use stack allocation, but capturing closures in defer can still allocate.

---

## 3. Why This Design?

### Explicit Over Implicit

Go’s function design prioritizes **explicitness** at every turn. Multiple returns force the caller to acknowledge all outputs, including errors. You cannot ignore an error without explicitly using `_`:

```go
result, _ := riskyOperation() // Explicitly ignoring error
```

This is philosophically aligned with Go’s rejection of exceptions—errors are just values that must be handled.

### No Function Overloading

Go does not support function overloading (multiple functions with the same name but different signatures). The designers considered overloading to create cognitive load: readers must examine call sites to know which implementation is invoked. Instead, Go encourages distinct function names or the use of variadic parameters.

**Aha moment:** Overloading seems convenient until you debug a `println` call that unexpectedly resolves to a different function because of integer promotion rules. Go’s simplicity trades a small amount of convenience for large gains in readability.

### Why Multiple Returns Instead of Tuples?

Languages like Python return tuples, which require unpacking. Java requires custom container classes or out parameters. Go’s multiple returns are syntactically lighter than tuples and **type-safe without runtime overhead**. The compiler verifies that the return count matches exactly.

### Named Returns as Documentation

Named returns serve primarily as **self-documenting code**. When a function returns multiple values of the same type (e.g., two `int`s), names clarify semantics. The second benefit—naked returns—is deliberately de-emphasized by the Go community because it reduces readability in long functions.

### Defer: Deterministic Cleanup Without Destructors

Unlike C++ destructors or Python’s `__del__`, Go’s `defer` provides **lexical, deterministic cleanup**. You pair acquisition and release in the same block, making resource management obvious. The trade-off is that you must remember to `defer`—the compiler does not enforce it.

---

## 4. Competing Approaches

| Language | Return Mechanism | Error Handling | Closure Model |
|----------|------------------|----------------|----------------|
| **Go** | Multiple returns | Values (`error`) | Captures heap-allocated context |
| **Python** | Single value (tuple) | Exceptions | Captures mutable cells (nonlocal) |
| **Java** | Single value | Exceptions (checked) | Must mark captured variables `final` or effectively final |
| **C++** | Single value (or out params) | Exceptions / `std::optional` | Captures by value or reference (user specifies) |
| **Rust** | Single value (`(T, U)`) | `Result<T, E>` (monadic) | Captures with move or borrow semantics |
| **JavaScript** | Single value (destructuring) | Exceptions / `throw` | Captures live scope (not just `final`) |

### Python
Python returns tuples and uses exceptions for errors. This leads to conventions like `value, error` being *possible* but not enforced. The `try/except` block separates error handling from return logic, which Go’s authors considered harmful for readability because errors can be silently ignored or caught too broadly.

### Java
Java’s single return type forces either:
- Throwing exceptions for error conditions (checked exceptions, which are widely criticized for causing empty catch blocks)
- Returning wrappers like `Optional<T>` or `Either<L,R>`
- Using out parameters (mutable arrays or container objects)

Java 21’s pattern matching improves destructuring but still lacks multiple return syntax.

### Rust
Rust’s `Result<T, E>` is a monadic type that must be unwrapped with `?` or `match`. This forces explicit error handling similar to Go, but with richer composition (e.g., `and_then`, `map_err`). Go’s designers rejected monadic error handling as too complex for most Go programmers, preferring the straightforward `if err != nil` pattern.

### C++
C++ supports multiple returns via `std::tuple` or structured bindings (C++17), but historically used out parameters (pass by reference). Out parameters obscure intent: `void divide(int a, int b, int& quotient, int& remainder)` does not clearly indicate which parameters are inputs vs outputs. Go’s multiple returns make outputs syntactically distinct.

### JavaScript
JavaScript’s destructuring allows pseudo-multiple returns: `const [q, r] = divide(10, 3)`. However, the function still returns a single array or object, incurring allocation overhead. Closures in JavaScript capture the entire scope chain, which can cause unintended memory retention.

**Key insight:** Go’s multiple returns are not merely syntactic sugar—they encode a **philosophical stance** that functions should be transparent about what they produce and what can go wrong, without requiring language-level monads or exception unwinding.

---

## 5. Common Mistakes

### Mistake 1: Shadowing Named Returns

Named returns are variables in scope. If you declare a new variable with the same name inside the function, you shadow the return variable.

```go
func shadowExample() (result int, err error) {
    // result is 0, err is nil
    if something {
        result, err := compute() // NEW variables, not the returns!
        if err != nil {
            return 0, err // returns 0, not the computed result
        }
    }
    return result, nil // result is still 0
}
```

**Fix:** Use assignment (`=`) instead of short declaration (`:=`), or avoid shadowing with distinct names.

### Mistake 2: Overusing Naked Returns

Naked returns (returns without arguments) reduce readability, especially in functions longer than a few lines.

```go
func badExample() (x, y int, err error) {
    // ... 20 lines of code
    return // What values are being returned? The reader must scan the entire function.
}
```

**Fix:** Explicit returns are idiomatic except for trivial getters or small helper functions.

### Mistake 3: Defer with Naked Return and Named Returns Gotcha

Deferred functions can read and modify named return values because they close over the function’s scope.

```go
func deferredModification() (result int) {
    defer func() {
        result++ // Modifies the return value!
    }()
    return 5 // Returns 6, not 5
}
```

This is sometimes intentional (e.g., logging final state), but often a source of bugs.

**Fix:** If you want to capture the final return value, use named returns. If you don’t, avoid modifying them in defer.

### Mistake 4: Closing Over Loop Variables

A classic closure trap across many languages: capturing the *same* loop variable in multiple closures.

```go
func loopTrap() {
    funcs := []func(){}
    for i := 0; i < 3; i++ {
        funcs = append(funcs, func() {
            fmt.Println(i) // Captures the same 'i' variable
        })
    }
    for _, f := range funcs {
        f() // Prints 3, 3, 3
    }
}
```

**Fix:** Capture a copy by passing `i` as an argument to the closure or creating a new variable inside the loop.

```go
for i := 0; i < 3; i++ {
    i := i // Shadow i with a new variable for this iteration
    funcs = append(funcs, func() {
        fmt.Println(i)
    })
}
```

Go 1.22+ fixes this for the `for` loop’s iteration variables, but the principle still applies to other cases.

### Mistake 5: Confusing Nil and Empty Slice in Variadic Functions

A variadic parameter receives a slice. If you call the function with no arguments, that slice is `nil`. If you call it with an explicit `nil` expansion, also `nil`. Many functions treat `nil` and empty slice identically, but not all.

```go
func log(prefix string, messages ...string) {
    if messages == nil {
        println(prefix, "no messages (nil)")
    } else if len(messages) == 0 {
        println(prefix, "no messages (empty)")
    } else {
        for _, m := range messages {
            println(prefix, m)
        }
    }
}

func main() {
    log("A")              // messages == nil, no messages (nil)
    log("B", []string{}...) // messages is empty slice, not nil
}
```

**Fix:** Always check `len(messages)` instead of comparing to `nil` unless you explicitly need to distinguish.

### Mistake 6: Defer in Loops

Defer executes when the *surrounding function* returns, not at the end of each loop iteration. This leads to resource accumulation.

```go
func loopDefer(files []string) error {
    for _, fname := range files {
        f, err := os.Open(fname)
        if err != nil {
            return err
        }
        defer f.Close() // All closes wait until function exits!
        // Process file...
    }
    return nil // Files remain open until after this line
}
```

**Fix:** Use an anonymous function to bind defer to a single iteration.

```go
for _, fname := range files {
    func() {
        f, err := os.Open(fname)
        if err != nil {
            // handle error within iteration
            return
        }
        defer f.Close()
        // Process file...
    }()
}
```

---

## 6. Performance Considerations

### Function Call Overhead

A direct function call in Go has **negligible overhead** (a few nanoseconds). However, calling a function through a **function value** (closure or function pointer) adds an indirect branch, which the CPU may mispredict. In performance-critical hot paths, storing functions in interfaces or using `switch` over function pointers can be faster.

### Inlining

The Go compiler aggressively **inlines** small functions (no loops, no complex control flow, under a certain byte size). Inlining eliminates call overhead and enables further optimizations across function boundaries.

```go
// This likely gets inlined
func add(x, y int) int { return x + y }

// This will not be inlined (contains loop)
func sumSlice(nums []int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}
```

You can see inlining decisions with `go build -gcflags="-m"`.

### Closure Allocation Cost

Every closure that captures variables **forces a heap allocation** for the captured environment. This allocation creates GC pressure. If you create many closures (e.g., in a loop), consider using a struct with methods instead.

```go
// Bad: allocates a closure per iteration
for i := 0; i < 1000000; i++ {
    fn := func() { process(i) } // heap allocation
    // use fn
}

// Better: pass context explicitly
type worker struct{ value int }
func (w worker) process() { /* use w.value */ }
```

### Defer Overhead

Deferred functions have measurable overhead, even after Go 1.14’s optimizations. In extremely hot code (e.g., per-request in a high-throughput server), avoid defer or use it only for expensive operations like mutex unlocks where the overhead is trivial relative to the operation.

```go
// Acceptable: mutex contention dominates
mu.Lock()
defer mu.Unlock()

// Unacceptable in hot path: avoid defer
func hotFunction() {
    start := time.Now()
    defer func() { recordLatency(time.Since(start)) }() // 100ns overhead per call
    // ... critical work
}
```

### Multiple Return vs. Container Allocation

Returning multiple values on the stack is **free**—no allocation. Contrast with languages that return a tuple or object (Python, JavaScript), which allocate heap memory.

```go
// Go: zero allocations
func stats(nums []int) (min, max, sum int) {
    // ...
}

// Python equivalent: allocates a tuple
def stats(nums):
    return (min_val, max_val, sum_val)  # tuple allocation
```

### Variadic Slice Allocation

When you call a variadic function with individual arguments, Go allocates a backing array for the slice. This allocation can be avoided by passing an existing slice with the `...` operator.

```go
// Allocates a new slice
println(sum(1, 2, 3, 4))

// No allocation (reuses the slice)
values := []int{1, 2, 3, 4}
println(sum(values...))
```

---

## 7. Best Practices (Idiomatic Go)

### Keep Functions Small and Focused

A function should do one thing and do it well. If you need to write a comment explaining “and” in the description (“validates input and writes to database and sends email”), split it.

### Use Named Returns Sparingly

Prefer named returns for:
- Functions with multiple return values of the same type (e.g., `(min, max int)`)
- Functions where a zero value return has special meaning (e.g., `(found bool, value int)`)
- Documentation in public API functions

Avoid naked returns except for trivial functions ≤3 lines.

### Always Handle Errors Explicitly

Never ignore an error without an explicit `_`. If you truly intend to ignore it, comment why.

```go
// Good: explicitly discarding, with justification
_ = file.Close() // Best effort; nothing we can do if it fails.

// Bad: ignoring without comment
file.Close()
```

### Defer for Cleanup, Not for Logic

Use `defer` for releasing resources (file handles, mutexes, network connections). Avoid using `defer` to modify return values or wrap logic—it obscures control flow.

### Prefer Returning Structs Over Many Return Values

If a function returns 4+ values, consider returning a struct. This improves readability and allows adding fields without breaking call sites.

```go
// Questionable: too many returns
func analyze(data []byte) (min, max, avg, median, mode float64, err error)

// Better
type Analysis struct {
    Min, Max, Avg, Median, Mode float64
}
func analyze(data []byte) (Analysis, error)
```

### Use Function Values for Strategy Pattern

Go’s first-class functions enable dependency injection without interfaces.

```go
type Processor struct {
    transform func(int) int
}

func (p *Processor) Process(nums []int) []int {
    result := make([]int, len(nums))
    for i, n := range nums {
        result[i] = p.transform(n)
    }
    return result
}

// Usage
p := &Processor{transform: func(n int) int { return n * 2 }}
```

### Document Function Behavior with Examples

Use `//` comments for godoc. Include a `// Example:` block for complex functions.

```go
// Sum returns the sum of all integers. If no arguments are provided,
// Sum returns 0.
//
// Example:
//   total := Sum(1, 2, 3) // total == 6
func Sum(nums ...int) int {
    // ...
}
```

---

## 8. Examples

### Example 1: File Processing with Multiple Returns and Defer

```go
package main

import (
    "bufio"
    "fmt"
    "os"
    "strconv"
    "strings"
)

// ReadIntegers reads a file where each line contains comma-separated integers.
// It returns two slices: valid integers and lines that failed parsing.
func ReadIntegers(filename string) (valid []int, invalid []string, err error) {
    file, err := os.Open(filename)
    if err != nil {
        return nil, nil, fmt.Errorf("opening file: %w", err)
    }
    defer file.Close()

    scanner := bufio.NewScanner(file)
    for lineNum := 1; scanner.Scan(); lineNum++ {
        line := strings.TrimSpace(scanner.Text())
        if line == "" {
            continue
        }
        
        parts := strings.Split(line, ",")
        for _, part := range parts {
            val, parseErr := strconv.Atoi(strings.TrimSpace(part))
            if parseErr != nil {
                invalid = append(invalid, fmt.Sprintf("line %d: %q", lineNum, part))
                continue
            }
            valid = append(valid, val)
        }
    }
    
    if err := scanner.Err(); err != nil {
        return valid, invalid, fmt.Errorf("scanning file: %w", err)
    }
    
    return valid, invalid, nil
}

func main() {
    valid, invalid, err := ReadIntegers("data.txt")
    if err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }
    
    fmt.Printf("Valid numbers: %v\n", valid)
    if len(invalid) > 0 {
        fmt.Printf("Failed to parse: %v\n", invalid)
    }
}
```

### Example 2: Retry Function with Closure and Variadic Options

```go
package main

import (
    "fmt"
    "time"
)

// RetryConfig holds optional parameters for retry behavior
type RetryConfig struct {
    MaxAttempts  int
    InitialDelay time.Duration
    MaxDelay     time.Duration
    Multiplier   float64
}

// DefaultRetryConfig provides sensible defaults
func DefaultRetryConfig() RetryConfig {
    return RetryConfig{
        MaxAttempts:  5,
        InitialDelay: 100 * time.Millisecond,
        MaxDelay:     5 * time.Second,
        Multiplier:   2.0,
    }
}

// Retry executes the given function up to MaxAttempts times with exponential backoff.
// It returns the function's result or the last error encountered.
func Retry(fn func() error, opts ...func(*RetryConfig)) error {
    // Apply functional options
    config := DefaultRetryConfig()
    for _, opt := range opts {
        opt(&config)
    }
    
    var lastErr error
    delay := config.InitialDelay
    
    for attempt := 1; attempt <= config.MaxAttempts; attempt++ {
        lastErr = fn()
        if lastErr == nil {
            return nil
        }
        
        if attempt == config.MaxAttempts {
            break
        }
        
        time.Sleep(delay)
        delay = time.Duration(float64(delay) * config.Multiplier)
        if delay > config.MaxDelay {
            delay = config.MaxDelay
        }
    }
    
    return fmt.Errorf("after %d attempts, last error: %w", config.MaxAttempts, lastErr)
}

// Helper functions for functional options
func WithMaxAttempts(n int) func(*RetryConfig) {
    return func(c *RetryConfig) { c.MaxAttempts = n }
}

func WithMaxDelay(d time.Duration) func(*RetryConfig) {
    return func(c *RetryConfig) { c.MaxDelay = d }
}

// Example usage
func main() {
    attempt := 0
    err := Retry(func() error {
        attempt++
        fmt.Printf("Attempt %d\n", attempt)
        if attempt < 3 {
            return fmt.Errorf("transient failure")
        }
        return nil
    }, WithMaxAttempts(5), WithMaxDelay(2*time.Second))
    
    fmt.Printf("Result: %v\n", err) // Success after 3 attempts
}
```

### Example 3: Middleware Pattern with Function Composition

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"
)

// Handler defines a function type for request handlers
type Handler func(ctx context.Context, req interface{}) (interface{}, error)

// Middleware is a function that wraps a Handler
type Middleware func(Handler) Handler

// Chain applies multiple middlewares to a handler
func Chain(h Handler, middlewares ...Middleware) Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}

// Concrete middleware implementations
func LoggingMiddleware(logger *log.Logger) Middleware {
    return func(next Handler) Handler {
        return func(ctx context.Context, req interface{}) (interface{}, error) {
            start := time.Now()
            logger.Printf("Request: %v", req)
            resp, err := next(ctx, req)
            logger.Printf("Response: %v, error: %v, duration: %v", resp, err, time.Since(start))
            return resp, err
        }
    }
}

func TimeoutMiddleware(timeout time.Duration) Middleware {
    return func(next Handler) Handler {
        return func(ctx context.Context, req interface{}) (interface{}, error) {
            ctx, cancel := context.WithTimeout(ctx, timeout)
            defer cancel()
            
            done := make(chan struct{})
            var resp interface{}
            var err error
            
            go func() {
                resp, err = next(ctx, req)
                close(done)
            }()
            
            select {
            case <-done:
                return resp, err
            case <-ctx.Done():
                return nil, fmt.Errorf("timeout after %v", timeout)
            }
        }
    }
}

func main() {
    // Business logic
    handleGreeting := func(ctx context.Context, req interface{}) (interface{}, error) {
        name, ok := req.(string)
        if !ok {
            return nil, fmt.Errorf("expected string, got %T", req)
        }
        return fmt.Sprintf("Hello, %s!", name), nil
    }
    
    // Apply middlewares
    logger := log.Default()
    wrapped := Chain(handleGreeting,
        LoggingMiddleware(logger),
        TimeoutMiddleware(1*time.Second),
    )
    
    ctx := context.Background()
    resp, err := wrapped(ctx, "Gopher")
    if err != nil {
        fmt.Printf("Error: %v\n", err)
    } else {
        fmt.Printf("Response: %v\n", resp)
    }
}
```

---

## 9. Summary & Exercises

### Summary

Go functions are intentionally straightforward but powerful:
- **Multiple returns** eliminate the need for out parameters or exception handling, making error flow explicit.
- **Named returns** serve as documentation and enable naked returns (use sparingly).
- **Closures** capture variables by reference, heap-allocating the environment—be mindful in hot paths.
- **Defer** provides deterministic cleanup with LIFO ordering, but carries a performance cost.
- **Variadic functions** are syntactic sugar for slice parameters; passing existing slices avoids allocations.

The philosophical thread throughout: **explicitness over cleverness**. Go’s function design chooses readability and predictable performance over features like overloading or monadic error handling. This trade-off rewards engineers building reliable systems where debugging time outweighs initial writing time.

### Exercises

#### Exercise 1: Retry with Jitter and Circuit Breaker
Build a function `RetryWithCircuitBreaker(fn func() error, threshold int, timeout time.Duration) error` that:
- Retries failed calls up to `threshold` times with exponential backoff
- If failures reach `threshold`, the circuit breaker opens, immediately returning `ErrCircuitOpen`
- After `timeout`, the circuit breaker transitions to half-open, allowing one trial call
- On half-open success, close the circuit; on failure, reopen for another `timeout`

Implement the circuit breaker state machine using closures. Include comprehensive error types using `errors.Is`.

#### Exercise 2: Memoizing Function with TTL
Write a generic memoization decorator:
```go
func Memoize[K comparable, V any](fn func(K) V, ttl time.Duration) func(K) V
```
The returned function caches results for duration `ttl`. Use a map from `K` to a struct containing `value V` and `expiry time.Time`. Handle concurrent access safely (use `sync.RWMutex`). Demonstrate with an expensive computation (e.g., Fibonacci, database query simulation). Include a method to clear the cache.

#### Exercise 3: Function Pipeline with Error Short-Circuiting
Create a pipeline composition function:
```go
func Pipe[T any](fns ...func(T) (T, error)) func(T) (T, error)
```
`Pipe` returns a function that applies each `fn` in sequence, passing the output of one as the input to the next. If any `fn` returns an error, the pipeline stops and returns that error immediately (no further functions execute). Test with a pipeline that validates a string, sanitizes it, and logs it. Ensure type safety without reflection.

---

**Answer to "Aha!":** Multiple returns and `if err != nil` aren’t ceremony—they’re a deliberate design forcing you to confront every error exactly where it occurs. In languages with exceptions, it’s easy to write `try { ... } catch { }` and swallow everything. Go makes that impossible without an explicit `_`, and that changes how you think about failure.
