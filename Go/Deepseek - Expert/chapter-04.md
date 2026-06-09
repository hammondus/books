## Chapter 4: Variables, Constants, and Types

Go’s type system, variable declarations, and constants feel immediately familiar to anyone who has worked with a statically typed language—until they don’t. The differences are subtle but profound: there are no uninitialized variables, constants can be *untyped*, and type inference works hand-in-hand with a compiler that refuses to compile unused local variables. This chapter unpacks the machinery behind the familiar-looking syntax, contrasts it with C, JavaScript, Python, Zig, and Rust, and equips you to wield Go’s type system idiomatically.

---

### 1. Basic Usage

Go offers several declaration styles. Each has a specific place.

```go
// Package-level: var, const, type
var defaultTimeout = 30 * time.Second // type inferred as time.Duration

const maxRetries = 3 // untyped constant

type Config struct {
    Host string
    Port int
}

func main() {
    // Local: short variable declaration (most common)
    ctx, cancel := context.WithTimeout(context.Background(), defaultTimeout)
    defer cancel()

    // Explicit type when zero value is useful
    var buf bytes.Buffer
    buf.WriteString("hello")

    // Multiple return values unpacked
    f, err := os.Open("data.txt")
    if err != nil {
        log.Fatal(err)
    }
    defer f.Close()

    // Constants can be typed
    const typedTimeout time.Duration = 5 * time.Second
    _ = typedTimeout
}
```

- `var` declares a variable with an explicit type or an initializer that drives inference. At package level, `var` is required; inside functions, `:=` is idiomatic.
- `:=` is a *short variable declaration* that both declares and assigns. It infers the type from the right-hand expression and only works inside functions.
- `const` declares a named constant. Constants can be character, string, boolean, or numeric values. They are compile-time expressions; you cannot, for instance, call `time.Now()` in a constant.
- Zero values are assigned automatically when a variable is declared without an initializer. Every type has a well-defined zero value: `0` for numerics, `""` for strings, `nil` for pointers/slices/maps/channels/interfaces, and a zeroed struct for structs.

Explicit numeric types (`int8`, `uint64`, `float32`, `complex128`) exist alongside machine-dependent `int` and `uint` (32 or 64 bits depending on architecture). The `byte` alias for `uint8` and `rune` alias for `int32` are used for character data.

---

### 2. Under the Hood

The compiler enforces strict rules that affect how variables and constants behave at runtime.

**Type inference:** When you write `x := 42`, the compiler performs *type unification*. It scans the right-hand side and assigns a default type: `int` for an integer literal, `float64` for a floating literal, `complex128` for complex, `string` for a string literal, and `bool` for boolean. This default type drives the inferred type of the variable. If context demands a different type (e.g., assignment to a `float32` field), the literal will be used as an untyped constant first, and then converted.

**Constants are not variables:** The Go spec draws a sharp line. Constants exist only at compile time. Numeric constants have *arbitrary precision*—they are represented as `big.Rat`-like values inside the compiler and never overflow until assigned to a typed variable. That’s why `const huge = 1 << 500` is legal, but `var hugeInt int = huge` fails.

**Zero-value initialization:** The compiler guarantees that all memory allocated for variables is zeroed. For a local variable that does not escape to the heap, the zeroing is a runtime operation: the compiler inserts instructions to set the stack frame to zero. For heap-allocated variables, the runtime zeroes the memory during allocation (`mallocgc` with `needzero` flag). This eliminates the entire class of “uninitialized variable” bugs but comes with a small, measurable cost we’ll explore in Performance Considerations.

**Stack vs. heap and escape analysis:** Variable declaration style (`var` vs `:=`) does not influence allocation; it’s purely syntax. The compiler’s escape analysis decides whether a variable escapes to the heap. A local variable declared with `var` can stay on the stack if it doesn’t escape, just like one declared with `:=`.

**Unused variables:** The Go compiler refuses to compile a program that has an unused local variable or an unused package import. This is a hard error at compile time. Package-level unused variables are permitted (they may be used by init functions or reflection), but the linter `go vet` will complain. This forces a cleanliness that removes dead code at inception.

---

### 3. Why This Design?

Go’s variable and type philosophy stems from the core design goal: *simplicity that scales to large codebases and large teams*.

**Zero values instead of uninitialized memory:** C and C++ leave variables uninitialized by default, leading to security vulnerabilities and heisenbugs. Go requires every variable to have a predictable starting state. The zero value is often useful directly: a `sync.Mutex` is usable without a constructor, an empty `bytes.Buffer` is ready to write, and a zeroed struct can often serve as a valid configuration. This eliminates the need for explicit constructors in many cases and reduces boilerplate.

**Why no implicit numeric conversions?** Go’s designers observed that automatic coercion (e.g., C’s promotion of `int` to `float`, JavaScript’s “==” shenanigans) was a constant source of subtle bugs. Go requires explicit conversion for all numeric types: you cannot assign an `int` to an `int32` without a cast. This verbosity forces you to think about loss of precision and signedness, and makes code intentions obvious at review time.

**Untyped constants:** By making constants untyped, Go gives them the flexibility of a macro without the dangers. A constant like `42` can be used as `int`, `float64`, `int64`, or even a custom `type Celsius float64` without explicit conversion, as long as the constant is representable in the target type. The moment you assign it to a variable, it becomes bound to that variable’s type. This design allows numeric literals to flow through code with less syntactic noise while keeping variables strictly typed.

**Short variable declaration (`:=`) vs. `var`:** Inside a function, `:=` is concise and reduces visual clutter. It also prevents accidental reliance on the zero value when you actually want an initialized value. At package level, the grammar forbids `:=` because package-level initialization order is more nuanced; `var` and `init()` handle it. This distinction is a conscious trade-off: a little grammar complexity for a lot of day-to-day readability.

**No null type, but `nil` is typed:** Go’s `nil` is not a universal null like JavaScript’s `null` or Python’s `None`. `nil` is a predeclared identifier that represents the zero value for pointers, maps, slices, channels, and interfaces. A `nil` pointer to a struct is not the same as a `nil` interface—an interface value has both a dynamic type and a dynamic value, and an interface holding a `nil` pointer is *not* `nil`. This design prevents the “billion-dollar mistake” of a single null everywhere but introduces the `nil` interface gotcha, which we will address in Common Mistakes.

---

### 4. Competing Approaches

Understanding Go’s choices becomes sharper when placed alongside its neighbors.

**C:** C variables are uninitialized by default (unless `static` or globals, which are zeroed). C has implicit conversions between numeric types, pointer arithmetic, and `void*` that erase type information. Go’s zero values and explicit conversions remove entire categories of undefined behavior. C has no type inference; you must declare types explicitly. Go’s `:=` removes drudgery but retains static safety.

**JavaScript:** JavaScript’s variables (when declared with `let` or `const`) are block-scoped but have `undefined` as the default if not initialized. Type coercion is pervasive. Go’s static typing with zero values eliminates runtime “undefined is not a function” errors. JavaScript’s `const` is a binding that cannot be reassigned, but the value may be mutable; Go’s `const` is a compile-time constant, fundamentally different.

**Python:** Python uses dynamic typing; variables are just names bound to objects. There are no type declarations, and “uninitialized” means a `NameError` at runtime. Type hints (PEP 484) add static analysis but are not enforced. Go’s approach gives the safety of static types at compile time while reducing ceremony with `:=`. Python’s “None” is a singleton object; Go’s zero values are type-specific, which gives more granular control (e.g., a `0` for an int is a valid value, not a sentinel).

**Zig:** Zig gives fine-grained control over initialization with `undefined` and compile-time checks to detect use of undefined variables. Zig also supports type inference with `const` and `var`. The key difference: Zig’s `undefined` is an explicit, potent tool that allows the programmer to bypass initialization for performance, while Go enforces zeroing always. Go chooses safety; Zig gives the expert an escape hatch. Zig’s `comptime` vastly expands what constants can be; Go’s constants are simpler but less powerful.

**Rust:** Rust’s `let` infers types, and variables are immutable by default; mutability must be declared with `mut`. Rust’s ownership system prevents use-after-move and enforces initialization before use at compile time. Both languages avoid uninitialized memory bugs, but through different means: Go zeroes everything at runtime, Rust proves initialization at compile time. Rust’s `const` and `static` have precise semantics around memory addresses and evaluation; Go’s constants are untyped and flexible. Rust’s `Option<T>` and `Result<T,E>` make the absence of a value explicit; Go uses zero values and `nil` which require discipline but less type machinery.

---

### 5. Common Mistakes

The seasoned engineer coming to Go will trip over these subtle traps.

**The “nil interface” gotcha:** An interface variable is nil only when *both* its dynamic type and dynamic value are nil. A concrete type pointer assigned to an interface makes the interface non-nil, even if the pointer is nil.

```go
var p *int = nil
var i any = p
fmt.Println(i == nil) // false!
```

The interface holds the type `*int` and value `nil`. To check if the underlying value is nil, you must reflect or use a type assertion. This is a frequent cause of unexpected panics when a nil pointer is stored in an error interface and later dereferenced.

**Shadowing with `:=`:** In a new block, `:=` declares new variables that shadow outer ones. This can hide an error variable and lead to ignoring a non-nil error.

```go
f, err := os.Open("a.txt")
if err != nil {
    return err
}
var data []byte
if data, err := io.ReadAll(f); err != nil { // new err shadows outer
    return err // returns nil because outer err is still nil!
}
```

The inner `err` is a fresh variable; the outer `err` is never updated. Use `=` if you intend to reuse the same variable, or check shadowing with `go vet`.

**Ignoring the zero value as a sentinel:** Because zero values are valid (e.g., `0` for an int), you cannot use them to represent “not set.” A map lookup returns the zero value for missing keys. You must use the comma-ok idiom:

```go
val, ok := m["key"]
if !ok {
    // key absent
}
```

**Constant overflow on assignment:** An untyped constant that exceeds the target type’s range causes a compile-time error, but a constant expression that overflows *during* constant evaluation can be subtle. For instance, `const tooBig = int(1 << 64)` fails; `const huge = 1 << 64` works until you try to assign it to a 64-bit int variable.

**Assuming `int` is 32 or 64 bits:** Go’s `int` size is platform-dependent. If you serialize an `int` directly, you may get different lengths on 32-bit vs 64-bit systems. For portable code, use fixed-width types like `int64` for numeric data that crosses process boundaries.

**Short declaration outside functions:** It’s a syntax error to use `:=` at package level. All package-level variables must use `var`. New Gophers often try `x := 5` at the top of a file.

---

### 6. Performance Considerations

Variables and constants themselves don’t add runtime overhead beyond what the underlying values require, but some patterns affect performance.

**Zero-value initialization cost:** When a goroutine’s stack grows or a new heap object is allocated, the runtime zeroes the memory. For large structs or arrays, this can be a visible cost. A `var arr [1e7]int64` on the heap will take time proportional to 80 MB of zeroing. In high-throughput allocation scenarios, reusing objects via `sync.Pool` or simply reducing allocations is more impactful than worrying about zeroing itself. The compiler cannot skip zeroing unless it can prove the entire variable is overwritten before use, which escape analysis rarely does for heap-allocated objects.

**Type inference at compile time:** `:=` and `var x = expr` resolve types during compilation. There is no runtime cost for inference. The compiler generates the same code for `var x int = 42` and `x := 42`. So choose the style that reads better.

**Escape analysis and variable placement:** The declaration style (`var` vs `:=`) does not affect escape decisions. A variable escapes if its address is taken and passed outside the function, or if it’s stored in a global or an interface. For performance-critical paths, avoiding indirections (pointers, interfaces) helps keep variables on the stack, which is cheaper to allocate and zero, and avoids GC pressure. Use `go build -gcflags="-m"` to see escape analysis output.

**Constants and binary size:** Constants are embedded directly into the instruction stream (or data sections for large constants). They contribute to binary size only as much as the values they represent. Overuse of large constant arrays could inflate the binary, but for typical numeric and string constants, the effect is negligible.

**Struct field alignment:** Although not strictly a variable declaration issue, the order of fields in a struct affects memory padding and cache line usage. The zero-value of a struct of the same logical fields but different ordering has different size. Use `unsafe.Sizeof` and `aligncheck` to optimize space, but in most cases, it’s a micro-optimization.

---

### 7. Best Practices

Idiomatic Go uses clear, consistent patterns for variables and constants.

- **Use `:=` for local variables** when you have an initializer. It reduces repetition and signals that the variable is being born with a meaningful value.
- **Use `var` for zero values** when the zero value is useful and explicit initialization would be redundant: `var buf bytes.Buffer`, `var mu sync.Mutex`.
- **At package level, always use `var` or `const`.** Group related declarations in `var` blocks for readability.
- **Name constants with CamelCase or UPPER_CASE?** Go convention: exported constants are `MixedCaps`, unexported `mixedCaps`. ALL_CAPS is not idiomatic unless it’s a legacy from generated code. Follow the standard library: `http.StatusOK`, `os.O_RDONLY`.
- **Prefer untyped constants for numeric sentinels.** They are more flexible. Use typed constants when the type is part of the constant’s meaning and you want to prevent accidental use in other contexts (e.g., `type contextKey string`).
- **Avoid using `nil` for sentinel “not found” when zero value is ambiguous.** Use a pointer, a separate boolean, or a wrapper type. For maps, always use the comma-ok idiom.
- **Check for shadowed variables with `go vet`.** The `shadow` analyzer (available via `golang.org/x/tools/go/analysis/passes/shadow`) catches accidental shadowing. Integrate it into CI.
- **Be explicit about fixed-width integers for I/O or cross-platform data.** Use `int64` for timestamps, `uint32` for sizes that must not exceed 4 GiB, etc. Reserve `int` for general counting and indexing.
- **When iterating, use `:=` to avoid aliasing bugs.** Classic closure over loop variable: before Go 1.22, the loop variable was reused; `for _, v := range items { go func() { use(v) }() }` captured the same `v`. Go 1.22 fixed this by making each iteration have its own variable, but older code still needs `v := v`. Be aware and rely on the new semantics if you control the toolchain version.

---

### 8. Examples

**Example: Configuration struct with zero values**
```go
type ServerConfig struct {
    Host    string        // defaults to ""
    Port    int           // defaults to 0
    Timeout time.Duration // defaults to 0s
}

func NewServer(cfg ServerConfig) *Server {
    if cfg.Host == "" {
        cfg.Host = "localhost"
    }
    if cfg.Port == 0 {
        cfg.Port = 8080
    }
    if cfg.Timeout == 0 {
        cfg.Timeout = 30 * time.Second
    }
    return &Server{cfg: cfg}
}
```
The zero value of `ServerConfig` is meaningful; we only override fields that are still zero. This is a lightweight alternative to constructors with many parameters.

**Example: Untyped constant flexibility**
```go
type Celsius float64
const boilingC Celsius = 100

const absoluteZeroC = -273.15 // untyped

var freezingF float64 = absoluteZeroC * 9/5 + 32 // works, untyped constant used as float64
var freezingC Celsius = absoluteZeroC              // works, assignable because untyped
```

If `absoluteZeroC` were typed as `float64`, the assignment to `Celsius` would require an explicit conversion. Untyped constants make type-defined scalars feel natural.

**Example: Avoiding shadow bug**
```go
func process(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return fmt.Errorf("open: %w", err)
    }
    defer f.Close()

    var result Result
    if result, err = parse(f); err != nil { // reuse err with =
        return fmt.Errorf("parse: %w", err)
    }
    // use result
    _ = result
    return nil
}
```
Using `=` instead of `:=` ensures the outer `err` is updated.

---

### 9. Summary & Exercises

**Summary:**
- Go’s variable declarations (`var`, `:=`) are clear and purpose-driven. `:=` is for local, initialized variables; `var` shines for zero values or package-level declarations.
- Constants are compile-time, untyped by default, and have arbitrary precision. They enable flexible numeric expressions without sacrificing safety.
- Zero values guarantee predictable state without constructors, eliminating uninitialized memory and reducing boilerplate.
- The design rejects implicit conversions and universal null, trading some initial surprise for code that behaves consistently under maintenance.
- Common pitfalls include the nil interface, variable shadowing, and assuming `int` width. Strong tooling (`go vet`) catches most of them.

**Exercises:**

1. **Design a Config Validator:** Write a function `Validate(cfg *Config) error` where `Config` is a struct with fields `Port int`, `Timeout time.Duration`, and `LogLevel string`. Use zero values to supply defaults: port 8080, timeout 30s, log level “info”. However, if the user explicitly set a value to the zero value of its type (e.g., they really want port 0), you must distinguish that from an unset field. How would you modify the design to allow distinguishing “not provided” from “provided but zero”? Implement your approach and discuss trade-offs.

2. **Shadowing Detective:** The following code compiles but contains a subtle bug. Identify it, explain the mechanism, and rewrite the code to fix it without altering the intended logic.
```go
func fetchAndCache(url string) ([]byte, error) {
    var data []byte
    if cached, ok := cache.Load(url); ok {
        data, ok := cached.([]byte)
        if !ok {
            return nil, fmt.Errorf("bad cache type")
        }
        return data, nil
    }
    resp, err := http.Get(url)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    data, err = io.ReadAll(resp.Body)
    if err != nil {
        return nil, err
    }
    cache.Store(url, data)
    return data, nil
}
```

3. **Zero-Cost Constants:** Create a small program that uses an untyped constant to represent the maximum payload size for a network packet (e.g., 1500 bytes). Use that constant in expressions with `int`, `int64`, and `uint32` variables without explicit conversions. Now intentionally write a constant expression that will overflow when assigned to a specific integer type and observe the compiler error. Explain why Go’s constant handling avoids runtime overflow checks in these expressions.

Each exercise should be accompanied by a short explanation of the solution and a reflection on how it embodies Go’s philosophy of simplicity and safety.
