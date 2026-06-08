## Chapter 5: Functions

Functions are the central unit of abstraction in Go. They carry no ceremonial baggage—no overloads, no default arguments, no traditional exception hierarchy. Instead, Go gives you a small, surgically precise set of tools: multiple return values, named result parameters, variadic signatures, closures, and deferred execution. This chapter unpacks each of them, shows how they map to the runtime, and explains why the Go team chose this path over the alternatives.

---

### 1. Basic Usage

A function declaration begins with the `func` keyword, an optional receiver (for methods), a name, a parameter list, and a return type or list of types. If multiple consecutive parameters share a type, you may collapse them.

```go
// Single return
func Add(x, y int) int {
	return x + y
}
```

**Multiple return values**—Go’s signature feature for error handling—work by listing types inside parentheses:

```go
func Divide(a, b float64) (float64, error) {
	if b == 0 {
		return 0, fmt.Errorf("division by zero")
	}
	return a / b, nil
}
```

You can name the result parameters. Named returns serve as documentation and allow “bare” returns, which implicitly return the current values of those variables.

```go
func OpenConfig(path string) (f *os.File, err error) {
	f, err = os.Open(path)
	if err != nil {
		// Wrap the error and return; f will be nil
		err = fmt.Errorf("open config: %w", err)
		return // bare return
	}
	// … further initialization
	return
}
```

**Variadic functions** accept a variable number of trailing arguments. The variadic parameter is a slice of the specified type:

```go
func Sum(nums ...int) int {
	total := 0
	for _, n := range nums {
		total += n
	}
	return total
}
```

Callers may pass individual arguments (`Sum(1,2,3)`) or spread an existing slice with `...` (`Sum(slice...)`).

**Anonymous functions and closures** are first-class values. They capture variables from the enclosing scope by reference:

```go
func Counter(start int) func() int {
	return func() int {
		start++       // captures 'start' by reference
		return start
	}
}
```

**Deferred calls** (`defer`) schedule a function to execute when the surrounding function returns, regardless of which path is taken. They run in LIFO order and are indispensable for resource cleanup:

```go
func ReadFile(path string) ([]byte, error) {
	f, err := os.Open(path)
	if err != nil {
		return nil, err
	}
	defer f.Close()
	return io.ReadAll(f)
}
```

---

### 2. Under the Hood

Go compiles each function into a contiguous block of machine code with a well-defined calling convention. Let’s zoom in on what the compiler and runtime do.

#### Calling Convention and Multiple Returns

Go’s ABI (as of Go 1.17+) passes arguments and results in registers where possible. For the `Divide` function above, the two `float64` parameters go into XMM registers, and the two return values (`float64`, `error`) come back in an XMM register and an integer register (the `error` is a two-word interface: type pointer and data pointer). Multiple return values cost nothing extra compared to a single composite return—they simply use more registers or, for larger counts, spill onto the stack. The caller reserves space in its own frame for results that cannot fit in registers, and the callee writes them there. The compiler lays out the caller’s stack to accommodate the maximum return footprint across all calls in the function.

#### Named Returns and Memory

Named result parameters are syntactic sugar over zero-initialised stack slots. The compiler allocates them at function entry just like any local variable. A bare `return` emits a sequence of register/memory loads from those slots. The critical insight: named returns do not introduce hidden allocations; they merely label existing stack locations, making the code self-documenting and occasionally saving a few keystrokes.

#### Inlining

The compiler aggressively inlines small, non-recursive functions that meet certain cost heuristics. Inlining a function that returns multiple values works seamlessly: the caller’s intermediate results map directly to registers, eliminating the call overhead entirely. Since Go 1.12, mid-stack inlining allows inline of functions that contain non-inlinable calls, further improving throughput. Functions with `defer`, closures, or loops beyond a certain size are not inlined, which is worth remembering in hot paths.

#### Closures

A closure is a struct allocated on the heap (if it escapes the creating function) or on the stack (if it doesn’t). This struct holds a function pointer and the captured variables. For `Counter`, the compiler generates a type roughly like:

```
type closure_Counter struct {
    F    func() int
    start *int
}
```

When `start` is captured, it is moved to the heap automatically—escape analysis ensures correctness. Each call to `Counter` creates a new closure struct, allocating a fresh `int` on the heap and pointing `start` to it. If the closure does not escape (e.g., it is passed only to synchronous callers within the same scope), the compiler may keep the entire structure on the stack and avoid allocation.

#### Defer Implementation

Before Go 1.13, `defer` translated into a runtime call that pushed a record onto a linked list, incurring noticeable overhead. Since then, the compiler optimises many deferred calls into a simple stack-allocated frame where the arguments are evaluated at the `defer` site and the call is executed directly in the epilogue. If a function has multiple defers, the compiler arranges them in reverse order. In hot paths, the overhead of a `defer` is now comparable to an extra function call; it is no longer a reason to avoid idiomatic cleanup.

---

### 3. Why This Design?

Every design choice in Go’s function system flows from a single principle: **make the common case explicit and the control flow linear**.

#### Multiple Returns Instead of Exceptions

Exceptions create hidden control-flow paths. A function call may jump to a distant `catch`, unwinding the stack and potentially leaving resources dangling if not scrupulously protected. Go’s multiple returns make error handling a **value returned alongside the normal result**. The caller _must_ handle or deliberately ignore the error (with `_` or an explicit blank assignment). This linearity forces the programmer to confront failure at the point of the call, resulting in more robust and readable code. The Go team saw that in large codebases, the clarity of explicit error paths outweighs the brevity of exceptions.

#### Why No Function Overloading or Default Arguments?

Overloading introduces ambiguity. Which `Read` overload should be called when types can be implicitly converted? Default arguments obscure call sites; you must consult the definition to understand what value is in play. Go says: one name, one signature. If you need variants, create clearly named functions (`ReadFull`, `ReadAtLeast`) or use a configuration struct with zero-value defaults. This simplifies tooling, eliminates the overload resolution tax on compile times, and keeps code self-documenting.

#### Closures but No “Class”

Closures provide data + behaviour without a named type, complementing Go’s composition model. They are lightweight, explicit, and avoid the ceremony of defining a whole struct for a single callback. Yet Go deliberately omits class-based inheritance, encouraging interfaces and embedding instead. Functions that return closures are the Go way to encapsulate state with behaviour—think of them as the idiomatic replacement for objects that only need one method.

#### Bare Return Philosophy

Bare returns are a double-edged sword. The Go spec includes them because in short functions with named results, they eliminate redundant typing and draw attention to the _variable names_ that describe the result. However, they are easy to misuse in longer functions. The community has largely settled on a rule of thumb: use bare returns only when the function fits on a single screen and the result names are well chosen. The language trusts the developer’s judgement, leaving the choice open rather than removing the feature.

---

### 4. Competing Approaches

Let’s compare Go’s function design with other mainstream languages.

#### Python: Exceptions and Multiple Returns via Tuples

Python functions return a single object. To return multiple values, you pack them into a tuple and unpack at the call site. Error handling relies on exceptions. This leads to deep try/except blocks and encourages the “look before you leap” (LBYL) vs. “easier to ask forgiveness than permission” (EAFP) debate. Go’s explicit errors avoid that duality; there is no exception to catch, so the code path is always visible. Python’s tuple returns are syntactically similar but allocate an ephemeral tuple, whereas Go’s multiple returns are zero-cost at the ABI level.

#### Java and C#: Exceptions and Return Objects

Java uses checked and unchecked exceptions, forcing the caller to declare or catch them. This can pollute method signatures and lead to empty `catch` blocks. C# largely relies on unchecked exceptions and `Try` pattern methods with `out` parameters. Both languages allow returning complex objects, but that often means creating a custom class or tuple. Go’s multiple returns eliminate such boilerplate: you return two values without boxing them into an object. Moreover, Java’s checked exceptions introduce a versioning headache—changing a method’s exception signature breaks callers, whereas adding an extra return value in Go is a compilation error at every call site, requiring deliberate updates.

#### C++ and Rust: Variadic Templates, std::optional, and Result

C++ has function overloading, default arguments, and variadic templates. Go intentionally avoids that complexity. Rust uses the `Result<T,E>` enum and the `?` operator for error propagation—a monadic approach that composes well and avoids exceptions. Both Go and Rust share the philosophy that errors are values, but Rust forces the developer to destructure the enum, while Go trusts the `if err != nil` convention. Rust’s approach is more expressive (combinators like `map`, `and_then`), but Go’s is simpler—there is no `Result` type to import, no early-return operator to learn; it’s just an `if` statement. This simplicity aids large-scale code comprehension.

#### JavaScript: Callbacks, Promises, and Exceptions

JavaScript’s concurrency model historically relied on error-first callbacks (`(err, data) => {}`), which mirror Go’s multiple returns but with asynchronous callbacks. Promises and `async/await` reintroduce try/catch for errors, mixing both paradigms. Go’s goroutines and channels handle asynchrony with the same explicit error return pattern, preserving consistency between sync and async code. This unification is a deliberate design win.

---

### 5. Common Mistakes

Even experienced engineers new to Go can stumble over these subtleties.

#### Shadowing Named Returns

A named result parameter creates a local variable. If you accidentally redeclare it inside an inner block (with `:=`), you shadow it, and the outer variable remains at its zero value. A bare `return` will then return the zero value instead of your intended result.

```go
func GetValue() (value int, err error) {
    // err is nil, value is 0
    tmp, err := someFunc() // err shadows the named result err
    if err != nil {
        return // returns 0, the inner err's value is lost
    }
    value = tmp
    return
}
```

**Fix:** Use the `var` form or assignment (`err = ...`) when the result variable is already declared, or avoid bare returns in such functions.

#### Defer Argument Evaluation

Arguments to a deferred function are evaluated **immediately**, not when the deferred call runs. A classic mistake is deferring a call with a variable that changes later.

```go
func Process(f *os.File, header []byte) error {
    // header might be modified later
    defer writeHeader(f, header)
    // … modify header …
    return nil
}
```

The `header` slice passed to `writeHeader` is the one at the `defer` line. To capture the final state, pass a pointer or use a closure:

```go
defer func() { writeHeader(f, header) }()
```

#### Closure and Loop Variable Capture (Pre-1.22)

Before Go 1.22, the loop variable was reused across iterations, causing all closures to capture the same memory location. This led to classic “all goroutines use the last value” bugs. Go 1.22 changed the semantics so each iteration creates a new variable. If you target older versions, the workaround is to shadow the variable (`v := v`). Understanding this history helps when reading legacy code.

#### Misusing Variadic with Slice Expansion

Passing a slice to a variadic function with `...` does **not** create a copy; the variadic parameter is exactly the provided slice. Modifying the slice inside the function mutates the caller’s data. If you need isolation, copy the slice inside the function.

```go
func AppendValue(base []int, vals ...int) []int {
    base = append(base, vals...)
    vals[0] = 999 // mutates caller's slice if vals was passed via ...
    return base
}
```

#### Returning a Reference to a Local Variable

Surprisingly, this is **not** a mistake in Go. The compiler automatically moves the variable to the heap if its address escapes. So returning `&localVar` is perfectly safe and idiomatic. The common mistake is to worry about it needlessly; the cost is the heap allocation and GC pressure, which we’ll discuss in performance.

---

### 6. Performance Considerations

Function design impacts performance in measurable ways.

- **Multiple return values:** Zero overhead. They do not allocate; they use registers and the stack. No boxing or heap allocation occurs, even for large structs if they fit in registers.
- **Closures:** If a closure captures variables and escapes, the captured variables are heap-allocated. Each creation of the closure allocates the closure struct (typically one pointer per captured variable). In a hot loop, prefer passing state explicitly via function parameters rather than creating a closure each iteration.
- **Defer:** Current overhead is roughly equivalent to a function call plus a small atomic operation. In the innermost hot path, you may choose to release resources manually and avoid `defer`, but in the vast majority of production code, the readability gain outweighs the minor cost.
- **Variadic functions:** Calling a variadic function with individual arguments allocates a slice on the stack if the compiler can prove its size fits. Spreading a slice with `...` avoids that allocation, but shares the underlying array, which can be a bug or a performance feature. Choosing between `...int` and a slice parameter depends on whether you want the caller to pay the allocation or you want a mutable copy. For hot-path library code, consider providing both a variadic convenience function and a slice-accepting core function.
- **Inlining:** Small, simple functions are inlined, eliminating call overhead entirely. Functions that return multiple values can be inlined; the caller adjusts the register/stack mapping. Inlining also enables further optimisations like dead code elimination. Keep functions small and avoid unnecessary `defer` if you want the compiler to inline them aggressively.

---

### 7. Best Practices

Idiomatic Go flows from clear, explicit function design.

1. **Use explicit returns over bare returns in long functions.** Bare returns are acceptable only when the function body is a handful of lines and the named results clearly document intent. For anything longer, write out the return values to avoid confusion.

2. **Prefer `if err != nil` with early returns.** This is the guard clause pattern. It keeps the happy path unindented and linear. Never use `panic` for normal error handling.

3. **Leverage closures for resource management with `defer`.** Always close files, release locks, and cancel contexts immediately after acquisition using a deferred closure that captures the resource.

4. **Use function types to decouple behaviour.** In Go, a function type is a first-class citizen. Define a `type Handler func(req *Request) error` and pass implementations around. This achieves dependency inversion without interface bloat.

5. **Name result parameters when they add clarity.** For instance, `func Open(name string) (f *os.File, err error)` makes the returned values self-documenting. Avoid naming if the types alone are sufficient.

6. **Limit variadic parameters to one per function, and place them last.** This is enforced by the language anyway. If you need multiple variadic slices, pass them as slices explicitly.

7. **Keep functions small and with a single responsibility.** A function should ideally do one thing and be no longer than a screenful. This promotes inlining, testability, and readability.

8. **Never ignore an error silently.** Use `_` only when the error is truly irrelevant and a comment justifies it. Even then, consider logging it at debug level.

---

### 8. Examples

**A robust file reader with a closure-based retry:**

```go
type RetryConfig struct {
    Attempts int
    Delay    time.Duration
}

func Retry(conf RetryConfig, fn func() error) error {
    var err error
    for i := 0; i < conf.Attempts; i++ {
        if err = fn(); err == nil {
            return nil
        }
        time.Sleep(conf.Delay)
    }
    return fmt.Errorf("after %d attempts: %w", conf.Attempts, err)
}

func ReadConfig(path string) ([]byte, error) {
    var data []byte
    err := Retry(RetryConfig{Attempts: 3, Delay: 50 * time.Millisecond}, func() error {
        f, err := os.Open(path)
        if err != nil {
            return err
        }
        defer f.Close()
        data, err = io.ReadAll(f)
        return err
    })
    return data, err
}
```

**Variadic min/max using Go 1.21 built-ins:**

```go
func MinMax(first int, rest ...int) (min, max int) {
    min, max = first, first
    for _, v := range rest {
        min = min // built-in min uses a different signature;
        // using manual for illustration
        if v < min {
            min = v
        }
        if v > max {
            max = v
        }
    }
    return
}
```

**Pipeline of string transformations using closures:**

```go
type Transform func(string) string

func Compose(transforms ...Transform) Transform {
    return func(s string) string {
        for _, t := range transforms {
            s = t(s)
        }
        return s
    }
}

// Usage:
trim := strings.TrimSpace
upper := strings.ToUpper
pipeline := Compose(trim, upper)
result := pipeline("  hello go  ") // "HELLO GO"
```

**Named returns with `defer` to capture final outcome:**

```go
func ProcessFile(path string) (err error) {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer func() {
        closeErr := f.Close()
        if err == nil {
            err = closeErr // only overwrite err if no prior error
        }
    }()
    // process f...
    return nil
}
```

---

### 9. Summary & Exercises

**Summary**

- Functions are declared with `func`, support multiple return values that are zero-cost at the ABI level, and use explicit `error` returns to replace exceptions.
- Named returns and bare returns serve readability in small doses but require caution.
- Variadic functions, closures, and `defer` round out the function toolbox, all underpinned by clear compile-time and runtime semantics.
- The design philosophy favours linear control flow, explicit error handling, and minimal ceremony—making Go code predictable and easy to reason about at scale.

**Exercises**

1. **Thread-safe memoization with a single-flight pattern.** Write a function `Memoize(fn func(string) (interface{}, error)) func(string) (interface{}, error)` that returns a cached version of `fn`. Ensure that concurrent calls for the same key result in only one execution of `fn`. Use closures and a mutex.

2. **Resource checkout with defer.** Design a type `Pool` that manages a limited set of connections. Provide a method `Borrow(ctx context.Context) (conn Connection, release func(), err error)` where `release` must be deferred by the caller to return the connection to the pool. Use a buffered channel internally.

3. **Pipeline DSL with error handling.** Extend the `Compose` example above so that each `Transform` returns `(string, error)` and the pipeline aborts on the first error. Compare the readability and performance of a closure-based pipeline with an explicit loop calling each stage.
