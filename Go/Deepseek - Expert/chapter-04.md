# Chapter 4: Variables, Constants, and Types

In most languages, declaring a variable is trivial. In Go, the seemingly simple act of creating a variable reveals deep design decisions about safety, readability, and compilation strategy. This chapter dissects how Go’s type system and initialization semantics shape idiomatic code—and why the language has no room for uninitialized memory or implicit type conversions.

## 1. Basic Usage

Go offers four distinct ways to declare a variable, each optimized for a specific context. All declarations ultimately produce a typed, initialized memory location.

### Explicit `var` with Type

```go
var port int            // zero value: 0
var host string         // zero value: ""
var enabled bool        // zero value: false
var config *Config      // zero value: nil (for pointers, slices, maps, channels, functions, interfaces)
```

The type goes *after* the variable name—a deliberate departure from C/Java that improves readability for complex types.

### `var` with Initialization (Type Inference)

```go
var port = 8080         // inferred as int
var host = "localhost"  // inferred as string
```

Go infers the type from the right-hand side. This is not dynamic typing; the type is fixed at compile time.

### Short Variable Declaration (`:=`)

```go
port := 8080            // equivalent to var port = 8080
host, port := "localhost", 8081  // multi-variable declaration
```

Available only *inside* functions. The compiler requires at least one new variable on the left side. This is the most common form in idiomatic Go.

### Multiple Assignment

```go
// Swap two values (no temporary variable needed)
a, b := 10, 20
a, b = b, a             // a=20, b=10

// Function returns (value, error)
file, err := os.Open("data.txt")
if err != nil {
    return err
}
```

### Constants

Constants are compile-time values. They can be untyped (numeric constants are especially flexible) or explicitly typed.

```go
const timeout = 30          // untyped integer constant
const maxRetries int = 5    // typed constant
const (
    StatusOK = 200
    StatusNotFound = 404
)
```

Untyped constants delay type determination until necessary, allowing them to work with any compatible type without explicit conversion:

```go
const factor = 2.5
var count int32 = 10
result := factor * float64(count)  // factor becomes float64 here
```

### `iota` for Enumerations

```go
const (
    Read  = 1 << iota  // 1 << 0 = 1
    Write              // 1 << 1 = 2
    Execute            // 1 << 2 = 4
)
```

`iota` resets to 0 at each `const` block and increments per line.

## 2. Under the Hood

### Zero Values: Intentional Initialization

When you declare `var x int`, the compiler emits code that writes `0` to that memory location. This is not a default in the C sense (where uninitialized memory contains whatever bytes were there previously). Go’s runtime guarantees that *every* variable has a well-defined value.

The zero value for a struct recursively zeroes its fields:

```go
type Connection struct {
    addr string
    port int
    conn net.Conn  // nil
}
var c Connection  // c.addr == "", c.port == 0, c.conn == nil
```

This eliminates an entire class of bugs from uninitialized memory reads. The cost is a small initialization overhead, but the compiler optimizes away redundant zeroing when it can prove the variable will be immediately overwritten.

### Type Inference at Compile Time

When you write `x := 42`, the compiler performs type inference *during parsing*. The algorithm is straightforward: the type of `x` is the type of the right-hand expression. For untyped constants like `42`, the inference rules assign a *default type* (int for integers, float64 for floats, string for strings, bool for booleans).

```go
x := 42      // x is int (default integer type)
y := 3.14    // y is float64
z := 1 + 2i  // z is complex128
```

This is a compile-time transformation—no runtime type information is involved. The generated assembly for `x := 42` is identical to `var x int = 42`.

### Constant Evaluation

Constants are evaluated by the compiler’s constant folding engine, which implements arbitrary-precision arithmetic. This allows operations that would overflow at runtime to be caught at compile time:

```go
const big = 1 << 100          // fine in compile-time big integers
// var fail = 1 << 100        // compile error: shift count too large for int
```

When a constant is assigned to a typed variable, the compiler checks for overflow at the assignment point:

```go
const big = 1 << 60
var x int32 = big  // compile error: constant overflows int32
```

### Memory Layout

Local variables (inside functions) typically live on the stack unless they escape to the heap (escape analysis, covered in Chapter 19). The compiler assigns stack offsets and emits MOV instructions to initialize them to zero or the specified value. Global variables (`var` at package scope) reside in the data or BSS segment.

For `:=` inside a loop, the variable is allocated once and reassigned on each iteration (unlike JavaScript's `let` inside loops, which creates a new binding per iteration in some contexts).

## 3. Why This Design?

### No Uninitialized Variables

The Go team explicitly rejected C’s "performance above correctness" stance on uninitialized memory. In C, `int x;` leaves `x` with an indeterminate value—a notorious source of security vulnerabilities and non-deterministic bugs. Go forces a choice: either you provide an initial value, or you accept the zero value, which is *always* safe to read.

This aligns with Go’s philosophy of **explicit over implicit**. Zero values are not "magic defaults"—they are defined behavior that enables useful patterns. For example, a `sync.Mutex` starts in an unlocked state, ready for use without a constructor.

### Why Constants Are Untyped by Default

Most languages (C++, Java) require a type for every constant. Go’s untyped constants solve the numeric type hierarchy problem elegantly. Consider:

```go
const secondsPerDay = 86400
var duration time.Duration = secondsPerDay * time.Second
```

If `secondsPerDay` were typed as `int`, the multiplication would require an explicit conversion to `time.Duration`. By remaining untyped, it adapts to the operation’s context. This is particularly valuable because Go has no implicit numeric conversions—untyped constants are the *only* place where the language bends this rule.

### The `:=` Syntax: Convenience Without Confusion

Short variable declarations exist to reduce repetition without introducing dynamic scoping or new runtime behavior. Unlike Python or JavaScript, `:=` does *not* rebind existing variables in an outer scope unless they are in the same block. Each `:=` creates at least one new variable, preventing accidental overwrites.

The design choice to limit `:=` to inside functions reinforces a clear boundary: package-level declarations must be explicit (`var` or `const`). This makes package APIs self-documenting and grep-able.

### No Implicit Type Conversion

Go forces you to write `int32(x)` to convert from `int`. This verbosity is intentional: implicit conversions between numeric types have caused countless bugs in C, Java, and JavaScript. By making conversions explicit, Go ensures that loss of precision or sign changes are visible in code review.

## 4. Competing Approaches

### C

C’s variable declaration puts the type first: `int x = 42;`. Uninitialized variables contain garbage. Type inference is absent until C++11’s `auto` (limited). C has true `const` (read-only memory) but also `#define` macros that aren’t type-checked.

**Go’s advantage:** No garbage values, consistent syntax for complex types (e.g., `var f func() int` vs C’s `int (*f)(void)`). **Trade-off:** Go’s zero initialization has a tiny cost; C can defer initialization for performance.

### JavaScript

JavaScript variables (`let x = 42`) are dynamically typed. `x` can become a string later. Uninitialized `let x;` yields `undefined`. `const` in JS means the binding is immutable, but the value can be mutated.

**Go’s advantage:** Type stability catches category errors at compile time. Zero values (`0`, `""`, `nil`) are predictable, unlike `undefined` which propagates through arithmetic to `NaN`. **Trade-off:** JavaScript’s flexibility enables REPL-driven experimentation; Go requires a full compilation cycle.

### Python

Python variables are references to objects; `x = 42` creates an `int` object. No compile-time type checking. Type hints (PEP 484) are optional and ignored by the runtime.

**Go’s advantage:** Types are enforced and checked. Zero values exist (e.g., `int()` returns `0`), but custom classes don’t get automatic zero instances—you get `None` unless you define `__init__`. **Trade-off:** Python’s duck typing allows more generic code without generics (though Go now has generics). Go’s zero values for structs are more powerful than Python’s `None`-by-default.

### Zig

Zig has `var` and `const` with type inference: `var x: i32 = 42;` or `const x = 42`. Uninitialized variables are *undefined behavior* (like C)—Zig provides `undefined` to opt in explicitly. Zig also has `comptime` for compile-time execution.

**Go’s advantage:** No accidental undefined behavior. Go’s zero values are safe and usable. **Trade-off:** Zig’s `comptime` is more powerful than Go’s constant folding, enabling generic code and compile-time data generation. Zig prioritizes low-level control over safety.

### Rust

Rust requires initialization before use, enforced by a borrow checker: `let x: i32;` is illegal without `let x = 42;`. Constants use `const X: i32 = 42;`. Rust also has `let` bindings that shadow immutably by default (`let mut` for mutation).

**Go’s advantage:** Simpler mental model. No distinction between `let` and `let mut`—variables are mutable unless declared `const`. Zero values provide safe defaults without requiring constructors. **Trade-off:** Rust’s strict initialization catches more bugs (e.g., using a variable before assignment in complex control flow). Go accepts some programs where a variable might be conditionally uninitialized? Actually, Go also forbids reading uninitialized variables—the compiler tracks all paths. But Rust’s rules are more precise with pattern matching.

## 5. Common Mistakes

### Shadowing with `:=`

The short declaration creates a *new* variable in the innermost scope, even if an outer variable has the same name:

```go
x := 42
if true {
    x := 100  // new variable, shadows outer x
    fmt.Println(x) // 100
}
fmt.Println(x) // 42 (still original)
```

This mistake often occurs with error handling:

```go
// Wrong: creates a new 'err' variable, doesn't assign to the outer one
var err error
if _, err := os.Open("file"); err != nil {  // err is new here
    // ...
}
// outer err remains nil
```

**Fix:** Use assignment (`=`) instead of `:=` in the inner scope.

### Misunderstanding Type Inference with Numeric Constants

```go
size := 65535      // size is int
var ptr *int32 = &size  // compile error: cannot use &size (value of type *int) as *int32
```

The default type for an integer constant is `int`, which is architecture-dependent (32 or 64 bits). Explicitly specify the type when you need a specific size:

```go
size := int32(65535)
```

### Reusing `:=` with Existing Variables Incorrectly

The rule: `:=` requires at least one new variable on the left. This works:

```go
count, err := strconv.Atoi("42")    // both new
if err != nil {
    return
}
count, ok := strconv.Atoi("10")     // error: count already declared, no new variable
```

**Fix:** Use assignment for `count` and `:=` for `ok`:

```go
count, err := strconv.Atoi("42")
// ...
var ok bool
count, ok = strconv.Atoi("10")  // assignment, not declaration
```

Or use a temporary variable.

### Constant Overflow

```go
const tooBig = 1 << 1000  // fine (compile-time big int)
var x int = tooBig        // compile error: constant overflows int
```

The error occurs at the point of assignment, not at the constant declaration. This can be surprising when constants are defined in another package.

### Forgetting That `const` Values Are Inlined

Because constants are inlined at compile time, taking the address of a constant is impossible:

```go
const answer = 42
ptr := &answer  // compile error: cannot take address of answer
```

This differs from C, where `const` often creates read-only memory locations with addresses.

### Zero Value Misconceptions

New Go developers sometimes expect `var m map[string]int` to create an empty map ready for insertion. Instead, `nil` maps read like empty maps but panic on write:

```go
var m map[string]int
_ = m["key"]   // returns 0 (zero value for int)
m["key"] = 1   // panic: assignment to entry in nil map
```

The same applies to slices with `nil` (fine for reading and `append`, but not for indexing beyond length) and channels (`nil` channels block forever).

## 6. Performance Considerations

### Zero Initialization Cost

Zeroing memory is not free, but it is cheap: modern CPUs can clear 64 bytes per cycle. For most applications, the cost is negligible compared to the correctness benefits. However, in hot loops, redundant zeroing can matter:

```go
var buf [1024]byte
for i := 0; i < 1000000; i++ {
    var tmp [1024]byte  // zeroed 1 million times
    copy(tmp[:], buf[:])
    // ...
}
```

**Optimization:** Move the declaration outside the loop and manually reset only necessary fields.

### Type Inference Is Free

`x := 42` produces identical assembly to `var x int = 42`. There is no runtime type lookup or boxing. The compiler resolves the type once during compilation.

### Constants Reduce Memory Loads

A constant is inlined as an immediate operand in assembly where possible. Compare:

```go
const limit = 100
for i := 0; i < limit; i++ { ... }
```

Versus:

```go
var limit = 100
for i := 0; i < limit; i++ { ... }
```

In the second case, `limit` lives in memory (stack or data segment), and each iteration loads it from memory. The compiler *may* optimize it into a register, but constant inlining is guaranteed. For frequently accessed values, constants eliminate the memory read.

### `int` Size Implications

`int` is 32 bits on 32-bit architectures and 64 bits on 64-bit architectures. Using `int` for loop indices and slice lengths is natural, but be aware that converting between `int` and `int64` requires an explicit cast, which may generate a MOV instruction (no significant cost). However, storing an `int64` in a struct on a 32-bit system has alignment padding implications.

### Short Variable Declaration and Escape Analysis

`:=` doesn't affect escape analysis—only the value's usage matters. However, the convenience of `:=` can lead to accidental allocations if the right-hand side returns a pointer that escapes. Always consider whether a value can stay on the stack (see Chapter 19).

## 7. Best Practices

### Use `:=` Inside Functions

Prefer `:=` over `var` for local variables with explicit initialization. It’s shorter, clearer, and prevents accidental zero values when you meant to initialize.

```go
// Good
count := 0

// Acceptable when zero value is intentional
var count int
```

### Group Related `var` or `const` Blocks

```go
// Good
var (
    host     string
    port     int
    timeout  time.Duration
)

// Instead of scattered declarations
```

### Use Explicit Types When Zero Value Is Meaningful

```go
var mu sync.Mutex        // zero value is usable
var buf bytes.Buffer     // zero value is an empty buffer ready to write
```

But for configuration values where zero is invalid, use `:=` with initialization:

```go
// Wrong: port 0 is invalid
var port int  // zero value 0, but we need non-zero

// Correct
port := 8080
```

### Use `const` for Magic Numbers

```go
const maxConnections = 100
const defaultTimeoutSeconds = 30
```

Avoid "magic numbers" in code—constants document intent and enable single-point updates.

### Use `iota` for Enumerations with Explicit Start

```go
type LogLevel int

const (
    Debug LogLevel = iota
    Info
    Warn
    Error
)
```

For bit flags, `iota` paired with shifts is idiomatic:

```go
type Permissions int

const (
    PermRead  Permissions = 1 << iota  // 1
    PermWrite                         // 2
    PermExec                          // 4
)
```

### Prefer `int` for General-Purpose Integers

Unless you need a specific size for serialization, C interop, or memory layout optimization, use `int`. It minimizes casting and works naturally with slice indexing, `len()`, and `range`.

### Avoid `var _ = ...` for Side Effects

Some developers use `var _ = x` to silence "unused variable" errors during development. Instead, use the underscore identifier:

```go
result, _ := someFunc()  // intentionally ignore error
```

Or comment out the variable entirely until needed.

### Name Variables for Clarity, Not Brevity

Go encourages short variable names, but not at the expense of understanding. Inside a 2-line loop, `i` is fine. At package scope, `conn` is better than `c`. The scope length determines the name verbosity.

```go
// Good for short scope
for i, v := range items { ... }

// Bad for package scope
var c *Connection  // what is 'c'?
var conn *Connection  // clear
```

## 8. Examples

### Example 1: Zero Values in Action

```go
package main

import "fmt"

type Server struct {
    addr    string
    port    int
    handler func() // nil function
}

func main() {
    var s Server
    fmt.Printf("addr=%q, port=%d, handler==nil: %v\n",
        s.addr, s.port, s.handler == nil)
    // Output: addr="", port=0, handler==nil: true

    // The zero value is immediately usable (handler will panic if called)
    // But we can set fields later:
    s.addr = "127.0.0.1"
    s.port = 8080
}
```

### Example 2: Typed vs. Untyped Constants

```go
package main

import (
    "fmt"
    "time"
)

const untypedSeconds = 86400              // untyped integer
const typedSeconds int64 = 86400          // typed

func main() {
    // Untyped constant adapts to the required type
    var d time.Duration = untypedSeconds * time.Second
    fmt.Println(d) // 86400s

    // Typed constant requires explicit conversion
    // var d2 time.Duration = typedSeconds * time.Second  // compile error: mismatched types
    var d2 time.Duration = time.Duration(typedSeconds) * time.Second
    fmt.Println(d2)

    // Untyped numeric constants work with any numeric type
    const pi = 3.14159
    var f32 float32 = pi      // fine, pi becomes float32
    var f64 float64 = pi      // fine, pi becomes float64
    var i64 int64 = pi        // compile error: constant 3.14159 truncated to integer
}
```

### Example 3: Iota for Bit Flags

```go
package main

import "fmt"

type Permission uint8

const (
    PermRead  Permission = 1 << iota // 1
    PermWrite                        // 2
    PermExec                         // 4
)

func (p Permission) String() string {
    names := []string{}
    if p&PermRead != 0 {
        names = append(names, "read")
    }
    if p&PermWrite != 0 {
        names = append(names, "write")
    }
    if p&PermExec != 0 {
        names = append(names, "exec")
    }
    return fmt.Sprintf("%v", names)
}

func main() {
    var perms Permission = PermRead | PermExec
    fmt.Println(perms) // [read exec]

    // Adding a permission
    perms |= PermWrite
    fmt.Println(perms) // [read write exec]

    // Removing a permission
    perms &^= PermWrite
    fmt.Println(perms) // [read exec]
}
```

### Example 4: Common Shadowing Bug and Fix

```go
package main

import (
    "fmt"
    "strconv"
)

func badParse() {
    var value int
    for _, s := range []string{"42", "99"} {
        // BUG: This creates a new 'value' variable scoped to the loop
        value, err := strconv.Atoi(s)
        if err != nil {
            continue
        }
        _ = value // uses the inner value
    }
    // Outer 'value' remains 0
    fmt.Println(value) // 0
}

func goodParse() {
    var value int
    var err error
    for _, s := range []string{"42", "99"} {
        value, err = strconv.Atoi(s) // assignment, not declaration
        if err != nil {
            continue
        }
        fmt.Println(value)
    }
}

func main() {
    badParse()
    goodParse()
}
```

## 9. Summary & Exercises

### Summary

- Go variables are **always initialized**—either explicitly or to their **zero value** (never garbage).
- Type inference (`:=`) is **compile-time** and has zero runtime cost.
- Constants are **untyped** by default, enabling flexible use across numeric types.
- `iota` provides a concise enumeration generator within `const` blocks.
- No implicit type conversions; explicit casting is required and visible.
- Shadowing with `:=` is the most common variable-related bug.
- Performance impact of zero initialization and constant inlining is minimal but measurable in hot paths.

### Key Philosophical Takeaways

- **Safety first:** Zero values eliminate uninitialized memory bugs.
- **Explicit is better than implicit:** Type conversions are visible; `:=` still requires at least one new variable.
- **Simplicity over cleverness:** Four declaration forms, each with a clear role.

### Exercises

#### Exercise 1: Zero Value Explorer

Write a program that declares variables of the following types without explicit initialization: `int`, `float64`, `string`, `bool`, `[3]int`, `[]int`, `map[string]int`, `struct{ x int; y func() }`, `chan int`, `interface{}`. Print each variable’s value and use type assertions or reflection to confirm the zero value. Then, explain in comments why each zero value is safe to use (or not) without further initialization.

**Challenge:** For the `map` and `slice` zero values, attempt to read from them (e.g., `m["key"]`) and then write to them. Document which operations panic and why.

#### Exercise 2: Constant Conversion Safety

Create a package that defines a constant `MaxUint32 = 4294967295` (the maximum value of a 32-bit unsigned integer). Then write a function that attempts to assign this constant to variables of type `uint8`, `uint16`, `uint32`, `uint64`, `int32`, and `float64`. Use the `errors` package to catch compile-time overflow (you'll need to comment out the failing assignments). Then, write a runtime equivalent using `var` and `uint32(4294967295)`—what happens when you assign to `uint8` at runtime? Use `fmt.Sprintf` to capture the overflow behavior.

**Goal:** Understand the difference between compile-time constant overflow (fatal error) and runtime overflow (silent wraparound or truncation).

#### Exercise 3: Build a Type-Safe Bitmask System

Implement a permissions system for files using `iota` and bit flags. Define a type `Permission` as `uint8`. Create constants for `Read`, `Write`, `Execute`, and `Delete` (use 4 bits). Implement methods:
- `String() string` returning a human-readable representation like `"rwx"` for read+write+execute.
- `Set(p Permission)` (adds bits)
- `Clear(p Permission)` (removes bits)
- `Has(p Permission) bool`

Then, write a function `Validate(perms Permission) error` that returns an error if `Delete` is set without `Write` (can’t delete without write permission). Demonstrate usage in `main` with a `switch` that handles different permission combinations.

**Advanced:** Use Go’s `flag` package to accept a permission string from the command line (e.g., `-perms rwx`), parse it into a `Permission` value, and validate it. Handle errors with `fmt.Errorf` wrapping.

---

**Next Chapter:** Functions—multiple returns, defer, closures, and why Go rejected exceptions.
