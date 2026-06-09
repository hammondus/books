# Chapter 4. Variables, Constants, and Types

In most systems languages, the type system is a tool for the compiler. In Zig, it is a tool for the programmer. Understanding Zig's binding and type semantics is not a prerequisite for writing code — it is a prerequisite for writing *correct* code, because Zig hands you a scalpel and trusts you to know which way it cuts.

This chapter covers the mechanics that underpin every declaration in Zig: the distinction between `var` and `const`, the comptime-driven type inference system, the deliberate choice to make `undefined` a trap rather than a convenience, and the coercion rules that govern how values flow across type boundaries. Engineers arriving from C, Rust, Go, or Python will find both familiar patterns and sharp departures, and the reasons for those departures are worth understanding precisely.

---

## 1. Basic Usage

### Declarations: `var` and `const`

Every local binding in Zig is declared with either `var` (mutable) or `const` (immutable). There is no `let`, no `val`, no `final` — just two keywords with an unambiguous semantic contract.

```zig
const std = @import("std");

pub fn main() void {
    // const: the binding cannot be reassigned, and its value cannot be mutated
    // through this binding. The type is inferred as `comptime_int` here,
    // resolved to i32 through peer type resolution (covered in section 2).
    const max_retries: i32 = 5;

    // var: the binding can be reassigned. The type must be explicit if there
    // is no initialiser that allows inference.
    var counter: i32 = 0;

    while (counter < max_retries) {
        counter += 1;
    }

    std.debug.print("ran {d} iterations\n", .{counter});
}
```

Type annotations follow the identifier using `:type` syntax. The annotation is optional when the compiler can infer the type from the initialiser — but many experienced Zig programmers annotate anyway for the benefit of readers and tooling.

```zig
const threshold = 1024;        // inferred: comptime_int (resolves at use site)
const name = "zig";            // inferred: *const [3:0]u8
const flag = true;             // inferred: bool
var   index: usize = 0;        // explicit: usize (common for index variables)
var   buf: [256]u8 = undefined;// explicit: [256]u8, initialised to undefined
```

### The Integer Menagerie

Zig provides the full complement of fixed-width signed and unsigned integers, plus a pair of sized types that track the target architecture:

| Type        | Description                          |
|-------------|--------------------------------------|
| `i8`–`i128` | Signed, 8 to 128 bits                |
| `u8`–`u128` | Unsigned, 8 to 128 bits              |
| `isize`     | Signed pointer-width integer         |
| `usize`     | Unsigned pointer-width integer       |
| `comptime_int` | Arbitrary-precision compile-time integer |

Zig also permits **arbitrary-width integers**: `i3`, `u7`, `i17`, `u63` — any bit width from 0 to 65535. These are not merely convenience aliases; the compiler enforces their range and lays them out correctly in `packed struct` contexts (covered in Chapter 20).

```zig
const flags: u3 = 0b101;   // three bits: values 0..7
const nibble: u4 = 0xF;    // four bits: values 0..15
```

### Floating-Point Types

```zig
const pi: f64 = 3.141592653589793;
const small: f32 = 1.5;
const precise: f128 = 3.14159265358979323846264338327950288;
```

Available: `f16`, `f32`, `f64`, `f80` (x86 extended), `f128`. The `f80` type maps directly to the x87 80-bit extended precision format; on non-x86 targets it is emulated in software. The `comptime_float` type, analogous to `comptime_int`, holds arbitrary-precision floating-point literals at compile time.

### Block Expressions as Values

Zig has no ternary operator. Instead, blocks are expressions — any block can produce a value using a labelled `break`:

```zig
const std = @import("std");

pub fn main() void {
    const x: i32 = 10;

    // A labelled block expression. `blk` is the label; `break :blk` carries
    // the value out. The type of `result` is inferred from the break value.
    const result = blk: {
        if (x > 5) break :blk x * 2;
        break :blk x + 1;
    };

    std.debug.print("{d}\n", .{result}); // 20
}
```

This pattern replaces ternary operators, `if`-initialiser chains, and statement-expression hacks common in C and C++. The block is a first-class value-producing construct.

### File-Scope Declarations

At the top level of a Zig source file (which is also a module), all declarations are implicitly `comptime` and must use `const` or `var`:

```zig
// top-level constants are comptime-evaluated; their values must be
// comptime-known.
const version: u32 = 1;
const build_string = "release-1.0";

// top-level var is allowed but rare; it implies mutable global state.
// Prefer passing state explicitly through function parameters.
var global_counter: u64 = 0;
```

The compiler evaluates top-level `const` declarations at compile time. They are not stack variables; they have no runtime storage (unless their address is taken, at which point they are placed in the read-only data segment).

---

## 2. Under the Hood

### Comptime Types and Peer Type Resolution

When you write `const x = 42;`, the literal `42` has type `comptime_int` — an arbitrary-precision integer that exists only at compile time. The same is true for `3.14`, which has type `comptime_float`. These types are not runtime types; they have no ABI representation. They are the compiler's internal bookkeeping for untyped numeric constants.

The type is resolved through **peer type resolution**: when a `comptime_int` is used in a context that demands a concrete type (a function argument, an assignment to an annotated variable, an arithmetic expression with a typed operand), the compiler selects the smallest type that can represent the value without loss, subject to context constraints:

```zig
const a = 100;          // comptime_int
var   b: i32 = a;       // a resolves to i32 — peer resolution with b's annotation
var   c: u8  = a;       // a resolves to u8  — 100 fits in u8, fine
// var d: u8 = 300;     // compile error: 300 does not fit in u8
```

This is not Hindley-Milner inference. It is not unification across a type constraint graph. It is a local, deterministic, left-to-right resolution at each use site. There is no "inference engine" to reason about; there is a set of coercion rules applied mechanically.

**Peer type resolution in binary operations** works by finding the common type among all operands:

```zig
const x: i32 = 10;
const y: i64 = 20;
// const z = x + y;   // compile error: i32 and i64 are not peers
const z = @as(i64, x) + y; // explicit cast required
```

Unlike C, Zig **never** performs silent integer promotion. There is no integer promotion rule. You must be explicit. This eliminates an entire class of bugs around implicit widening and sign extension.

### How `const` Is Compiled

For local `const` bindings whose values are comptime-known (literals, comptime expressions, and functions of those), the compiler may fold them entirely — no stack slot, no load instruction. The value is embedded as an immediate operand in the generated machine code.

For local `const` bindings whose values are runtime-computed (a function return value, an I/O result), a stack slot is allocated exactly as for `var`, but the compiler emits no store instructions after initialization and can prove no alias escapes. This enables better alias analysis and, consequently, better optimizations from LLVM.

You can observe the difference by compiling with `zig build-exe -femit-llvm-ir`:

```
// var x: i32 = foo();         -- allocas, stores, loads
// const x: i32 = foo();       -- allocas, one store, loads potentially eliminated
```

### `undefined`: The Poisoned Value

`undefined` in Zig is not zero, null, or any other sentinel. In **Debug** and **ReleaseSafe** builds, Zig initialises `undefined`-tagged memory with the byte pattern `0xaa`. If you read from it, you get garbage data patterned as `0xaa...`; if you dereference a pointer sourced from it, you get a likely crash at a recognisable address rather than silent data corruption.

In **ReleaseFast** and **ReleaseSmall** builds, `undefined` is genuinely undefined — the compiler treats it as a free licence to assume the memory was never written, enabling dead-store elimination and other transformations. Reading from it in these modes is undefined behaviour in the C sense.

```
Debug/ReleaseSafe: var buf: [16]u8 = undefined;
  => memset(buf, 0xAA, 16)  (conceptually; may be implicit or explicit)

ReleaseFast/ReleaseSmall: var buf: [16]u8 = undefined;
  => no initialisation; contents are whatever the stack held
```

The `0xaa` pattern is recognisable in a hex dump (`0xAAAAAAAA` as a 32-bit value is `2863311530`), which is intentional: it produces obviously wrong outputs rather than plausible ones.

### Type Size and Alignment

Zig follows the platform ABI for alignment of primitive types. `@sizeOf(T)` and `@alignOf(T)` are comptime builtins that return the size and required alignment of a type in bytes:

```zig
const std = @import("std");

pub fn main() void {
    std.debug.print("i32:  size={d} align={d}\n", .{ @sizeOf(i32),  @alignOf(i32)  });
    std.debug.print("i64:  size={d} align={d}\n", .{ @sizeOf(i64),  @alignOf(i64)  });
    std.debug.print("bool: size={d} align={d}\n", .{ @sizeOf(bool), @alignOf(bool) });
    std.debug.print("u3:   size={d} align={d}\n", .{ @sizeOf(u3),   @alignOf(u3)   });
    // u3 has sizeOf=1 (smallest addressable unit), but is 3 bits logically.
    // Its alignment is 1. In a packed struct it would occupy exactly 3 bits.
}
```

---

## 3. Why This Design?

### Why Is There No Implicit Numeric Promotion?

C's integer promotion rules — where `char` and `short` silently widen to `int` before arithmetic, and where signed/unsigned mixing in comparisons produces counterintuitive results — are among the most persistent sources of subtle bugs in C codebases. Rust similarly requires explicit casting between numeric types. Zig makes the same choice, but for a different reason than Rust.

Rust requires explicit casts partly to satisfy the borrow checker's type system. Zig requires them because of the thematic pillar: **explicit is better than implicit**. Every type boundary that a value crosses must be visible in the source code. This is not inconvenience for its own sake — it means that a code reviewer scanning for numeric truncation can grep for `@truncate`, can grep for `@intCast`, and can audit every such site. If the cast is implicit and silent, the reviewer has no surface to audit.

The `comptime_int` / `comptime_float` escape hatch exists precisely to avoid making constants painful: literal `42` adapts to its context without a cast, because its adaptation is determined by the programmer who wrote the annotation, not by an invisible promotion rule.

### Why Two Keywords Instead of Immutability by Default?

Rust's `let` is immutable by default; mutability must be opted into with `let mut`. Go uses `:=` for inference and `var` for explicit declarations, with mutability being the default. Zig uses `const` and `var` as equals — neither is the default; you must always say which you mean.

The rationale is not aesthetic symmetry. It is **readability at the declaration site**. When you scan a function body, `const` is a strong signal: this value is fixed, do not look for modification. `var` is a flag: this value changes; track it. The explicit choice forces the programmer to think about mutability at declaration time rather than mutating first and adding `mut` only when the compiler complains (the common Rust workflow).

### Why Is `undefined` a Poison Rather Than Zero?

Several languages provide zero-value initialisation for all variables (Go, Java). Others require explicit initialisation before use (Rust's ownership system enforces this via the borrow checker). C leaves uninitialised variables as truly undefined. Zig stakes out a position deliberately between "zero by default" and "truly undefined":

- **Zero by default** (Go's approach) masks bugs. A `user_id` of zero is a valid `u64`; you may never notice that the field was never populated.
- **Truly undefined** (C's approach in release builds) leads to heisenbugs that appear and disappear depending on stack contents.
- **Poison value** (`0xaa` in debug): forces failures to be loud and recognisable early in development while giving the release optimizer maximum freedom.

You opt into zero-initialisation explicitly with `std.mem.zeroes(T)` or by writing `= .{};` (which zero-initialises a struct). The cost is visible; the decision is deliberate.

### Why No Type Inference for Mutable Variables Without an Initialiser?

You cannot write `var x;` and assign later. The type annotation is mandatory when there is no initialiser:

```zig
var count: usize = undefined; // valid: type known, value will be set later
// var count = undefined;     // compile error: type cannot be inferred from `undefined`
```

This is the same explicit-is-better-than-implicit logic. `undefined` carries no type information — it is a stand-in for "I will initialise this before use." The type must come from somewhere, and the programmer is the authoritative source.

---

## 4. Competing Approaches

### C: Implicit, Pervasive Mutation

In C, all local variables are mutable by default. `const` exists but is advisory; it applies to the binding, not to the full reach of the value:

```c
int x = 10;
const int *p = &x;  // p is a pointer to const int
*p = 20;            // compile error (via the pointer)
x  = 20;            // fine — x itself is not const
```

C has no concept of comptime types; all literals have fixed types (integer literals are `int` unless suffixed). Silent promotion rules (`char` → `int`, `float` → `double` in function calls) are pervasive. Uninitialised variables are truly undefined with no debug-mode protection.

Zig's improvements here are concrete and measurable: no silent promotion, explicit `undefined` poisoning in debug mode, and `const` that is enforced through all pointer paths.

### Rust: `let` Immutability with Borrow-Checked Mutation

Rust defaults to immutable bindings (`let x = 5;`) and requires `let mut x = 5;` for mutability. The borrow checker enforces these constraints transitively — you cannot obtain a `&mut T` from an immutable binding.

Rust's integer literals are also typed at the use site, with inference flowing from context. But Rust's inference is Hindley-Milner: it can propagate type constraints across multiple statements and infer a type from a use site that appears much later in the function. Zig does not do this; type resolution is local and unambiguous.

```rust
let mut v = Vec::new();
v.push(1i32);  // Vec<i32> inferred retroactively from the push
```

In Zig, this retroactive inference does not exist. `std.ArrayList` requires an explicit type parameter:

```zig
var list = std.ArrayList(i32).init(allocator);
```

This is not a limitation — it is a deliberate legibility choice. A reader never has to search forward in a function to understand what type a binding holds.

For numeric types, Rust requires explicit casts (`x as i64`) between all concrete integer types. Zig uses a family of comptime builtins (`@intCast`, `@truncate`, `@bitCast`, `@floatFromInt`) that are more granular and more self-documenting about what conversion semantics are intended.

### Go: Zero Values, No Constants Without Runtime Cost

Go zero-initialises all variables. A `var x int` starts at `0`; a `var s string` starts at `""`. This produces predictable, safe code but can mask bugs where a field was never intentionally set.

Go's `:=` syntax combines declaration, type inference, and assignment. It works only inside functions; package-level variables require `var`. Go has no equivalent of Zig's `comptime_int`; untyped constants in Go are a similar concept (Go spec calls them "untyped numeric constants"), but the inference rules differ.

Go has no arbitrary-width integers. The smallest integer type is `int8`. Zig's `u1`, `u3`, `i7` types are unavailable in Go, making certain bit-packing operations require manual masking.

### Python: Dynamic Typing and the Absence of the Problem

Python's `x = 10` dynamically binds the name `x` to an integer object. Mutability is a property of the object, not the binding — lists are mutable, tuples are not. There are no type annotations enforced at runtime (only hints for static analysis). The comparison is most useful to highlight what Zig adds: when you have no type system, you have no hidden coercions, but you also have no compile-time guarantees.

For engineers accustomed to Python's ergonomics, Zig's explicit type annotations will feel verbose at first. The payoff is that the type is documentation — a reader of `var offset: usize = 0;` knows immediately that `offset` is used for indexing into memory. `var offset = 0` in Python says nothing about the intended range.

### C++: `const`, `constexpr`, `auto`, and Their Interactions

C++ layered `constexpr` on top of `const`, producing a complex lattice of "evaluable-at-compile-time" guarantees with numerous edge cases. `auto` enables type inference but can lead to surprising deductions (`auto x = {1, 2, 3};` in C++ is an `initializer_list<int>`, not an array). The interaction between `const`, `constexpr`, `volatile`, `mutable`, and reference qualifiers is a regular source of confusion.

Zig collapses this: `const` means immutable, `comptime` means compile-time, and these are orthogonal. There is no `constexpr` because Zig's `comptime` is far more powerful and applies to entire functions, not just expressions.

---

## 5. Common Mistakes

### 1. Assuming `const` Means the Pointed-To Data Is Immutable

```zig
const std = @import("std");

pub fn main() void {
    var data: [4]u8 = .{ 1, 2, 3, 4 };

    // `ptr` is a const binding, but it points to mutable data.
    // This is a *const [4]u8 pointer — the pointer itself cannot be
    // reassigned, but can we mutate through it?
    const ptr = &data;
    // ptr[0] = 99;  // compile error: cannot assign through const pointer

    // To mutate through a pointer, both the binding and the pointee
    // must be mutable:
    var mutable_ptr: *[4]u8 = &data;
    mutable_ptr[0] = 99;
    std.debug.print("{d}\n", .{data[0]}); // 99
}
```

The type of `&data` when `data` is a `var` is `*[4]u8` — a mutable pointer. Assigning it to a `const` binding produces a `*const [4]u8` via implicit const coercion. You cannot write through a `*const T` pointer. This is the expected behaviour, but engineers from C sometimes expect `const ptr` to mean "const binding, mutable pointee." In Zig, `const` applied to a pointer binding makes the pointer itself immutable *and* makes the pointee immutable through that pointer.

### 2. Using `undefined` Without Writing Before Reading

```zig
pub fn broken() i32 {
    var x: i32 = undefined;
    // forgot to assign x
    return x * 2; // In Debug: returns 0xAAAAAAAA * 2 = garbage
                  // In ReleaseFast: truly undefined behaviour
}
```

The Debug-mode poisoning catches this most of the time, but only if you observe the result. Computations over `0xaa...` values rarely crash — they just produce wrong answers. Always initialise before use. For complex structs, use `std.mem.zeroes(T)` explicitly if zero-init is the right semantics; use `= .{}` for struct zero-initialisation; use `= undefined` only when you are about to write every field before the first read.

### 3. Forgetting That Shadowing Is Forbidden

Zig does not permit shadowing a variable from an outer scope with a new declaration in an inner scope:

```zig
pub fn example() void {
    const x: i32 = 5;
    {
        const x: i32 = 10; // compile error: redefinition of 'x'
        _ = x;
    }
    _ = x;
}
```

This catches a class of bug common in C and JavaScript where an inner `x` silently shadows an outer one. If you intend to use a different value, give it a different name. If you intend to rebind (which Zig allows for `var`), reassign rather than redeclare.

### 4. Expecting Implicit Numeric Coercion Across Types

```zig
pub fn add_wide(a: i32, b: i64) i64 {
    // return a + b; // compile error: i32 and i64 cannot be added without explicit cast
    return @as(i64, a) + b; // correct
}
```

Every numeric type boundary requires an explicit operation. The available casting builtins are:

| Builtin          | Semantics                                          |
|------------------|----------------------------------------------------|
| `@as(T, v)`      | Explicit type assertion/coercion (comptime-safe)   |
| `@intCast(v)`    | Runtime-checked narrowing; panics if out of range  |
| `@truncate(v)`   | Bit-truncation without range check (wraps)         |
| `@bitCast(v)`    | Reinterprets bit pattern (same bit-width required) |
| `@floatCast(v)`  | Float narrowing (precision may be lost)            |
| `@floatFromInt(v)` | Integer to float conversion                      |
| `@intFromFloat(v)` | Float to integer, truncates toward zero          |

The distinction between `@intCast` (checked, panics on overflow) and `@truncate` (unchecked, wraps) is critical. Use `@intCast` when overflow indicates a bug. Use `@truncate` when bit-masking is intentional.

### 5. `comptime_int` Leaking Into Runtime Code

```zig
const limit = 1000; // comptime_int

pub fn count_up(n: comptime_int) void { // intentional comptime parameter
    _ = n;
}

pub fn bad_runtime(n: i32) void {
    // const shifted = limit << n; // compile error: n is not comptime, limit is comptime_int
    // Fix: give limit an explicit type
    const typed_limit: i32 = 1000;
    const shifted = typed_limit << @intCast(n);
    _ = shifted;
}
```

`comptime_int` cannot participate in runtime arithmetic. It must be coerced to a concrete type before use with a runtime value. The error message from the compiler is usually clear, but the surprise is that a bare literal constant like `limit = 1000` is not already an `i32`.

### 6. Confusing `@as` With a Cast

`@as(T, v)` is an **assertion**, not a conversion. It tells the compiler "treat `v` as type `T`." It is valid only when the coercion is lossless and follows Zig's coercion rules (comptime-known in-range, pointer constness, peer types). It does not truncate, does not reinterpret bits, and does not perform float-to-int conversion. Misusing `@as` where `@intCast` is needed will produce a compile error:

```zig
const x: i64 = 1_000_000_000_000;
// const y = @as(i32, x); // compile error: i64 cannot be coerced to i32
const y: i32 = @intCast(x); // runtime-checked narrowing — will panic if > 2^31-1
```

---

## 6. Performance Considerations

### `const` Enables Better Alias Analysis

When the compiler knows a binding cannot be modified, it can freely hoist loads, elide redundant reads, and prove that two pointers do not alias — all without explicit `restrict` annotations. In hot loops where a constant bound is checked at each iteration, the compiler can eliminate the load entirely:

```zig
pub fn sum(slice: []const i32) i64 {
    const len = slice.len; // const: loaded once, hoisted by the compiler
    var total: i64 = 0;
    var i: usize = 0;
    while (i < len) : (i += 1) {
        total += slice[i];
    }
    return total;
}
```

With `const len`, the compiler can prove that `len` does not change across iterations and can keep it in a register. With `var len`, the compiler would need to prove the loop body does not modify it (which it can sometimes do via escape analysis, but this is not guaranteed).

### Zero-Initialisation Has a Cost

`std.mem.zeroes(T)` emits a `memset` or equivalent zeroing sequence. For large stack buffers this can be expensive. Prefer `= undefined` for large buffers you are about to fill completely, and reserve zero-initialisation for structures where you genuinely need the zero guarantee (network buffers, cryptographic state, etc.).

```zig
// Expensive: zeroes a 4KB buffer on every function call
var buf = std.mem.zeroes([4096]u8);

// Free: marks the buffer as undefined; you must fill it before reading
var buf: [4096]u8 = undefined;
// ... fill buf completely before using it
```

### Arbitrary-Width Integers Are Not "Free"

`u3` in a normal `var` declaration is stored in a `u8` (one byte, smallest addressable unit) at runtime. Its value is constrained to 0–7 but it occupies 8 bits of stack space. The arbitrary-width behaviour is only bit-exact inside `packed struct` (where contiguous `u3` fields pack into the minimum number of bytes) and in arithmetic operations (where the compiler generates masking code to enforce the range).

```zig
const std = @import("std");

pub fn main() void {
    var a: u3 = 7;
    var b: u3 = 1;
    const c = a +% b; // wrapping add: 7 + 1 = 0 (mod 8), not overflow panic
    std.debug.print("{d}\n", .{c}); // 0
    _ = b;
}
```

Using `+%` (wrapping add) vs `+` (checked add, panics on overflow in Debug mode) is a deliberate choice that the programmer makes visible in the source. There is no silent wrap.

### Comptime Constants Are Free

File-scope `const` values with comptime-known values incur zero runtime cost. They are folded into the instruction stream as immediates or placed in `.rodata`. The `comptime_int` / `comptime_float` mechanism exists precisely so that large numeric constants can be used freely without accidental storage overhead:

```zig
const buffer_size: usize = 64 * 1024;     // 65536 — an immediate in generated code
const default_timeout_ms: u32 = 30_000;   // 30000 — immediate
```

The underscore in `30_000` is a readability separator with no runtime meaning; the compiler strips it.

---

## 7. Best Practices

### Prefer `const` By Default

Reach for `const` first. Upgrade to `var` only when you know the value will change. This is not Rust's "immutable by default" enforced ergonomically — it is a discipline that makes function bodies scannable. A reader who sees only `const` declarations in a function body knows immediately that no state is being mutated; the function is, in that local sense, pure.

```zig
// Preferred: const for values that don't change
pub fn compute_checksum(data: []const u8) u32 {
    const len = data.len;
    var checksum: u32 = 0;
    for (data[0..len]) |byte| {
        checksum ^= byte;
    }
    return checksum;
}
```

### Always Annotate Types on Function-Scope `var` Declarations

Type inference on `var` is sometimes surprising, especially for integer literals. Be explicit at `var` declarations to avoid confusion and to make the type visible at the declaration site:

```zig
// ambiguous: what type is `count`?
var count = 0;

// clear: usize for loop indices and lengths
var count: usize = 0;
```

### Use `@as` to Document Intent, Not to Coerce

When a coercion is guaranteed by the type rules (e.g., coercing a `*[N]T` to a `[]T`), write it explicitly using `@as` to make the intent clear, especially in public APIs:

```zig
pub fn to_slice(arr: *[16]u8) []u8 {
    return @as([]u8, arr); // explicit, self-documenting coercion
}
```

### Initialise `undefined` Buffers Immediately

When you declare a buffer with `undefined`, add a comment explaining why, and initialise it before the first read. A common pattern in I/O code:

```zig
const std = @import("std");

pub fn read_line(reader: anytype) ![]const u8 {
    // Buffer is undefined here; filled by reader.readUntilDelimiter.
    // No bytes are read before the buffer is written.
    var line_buf: [1024]u8 = undefined;
    return reader.readUntilDelimiter(&line_buf, '\n');
}
```

### Use `_` to Suppress Unused Variable Warnings

Zig makes unused variables a compile error. Use `_ = variable;` as a no-op assignment to suppress the warning when you genuinely intend to ignore the value:

```zig
const result = compute();  // result is some type you don't need
_ = result;                // explicit discard; the reader knows it's intentional
```

Do not use `_ =` reflexively everywhere. If you are discarding a value frequently, reconsider the API design.

### Prefer Named Numeric Types Over Bare Literals in Struct Fields

When defining a struct (covered in Chapter 11), annotate all field types explicitly, even when default values are provided. This makes the struct's memory layout immediately readable:

```zig
const Config = struct {
    max_connections: u32 = 100,
    timeout_ms: u64 = 5000,
    debug: bool = false,
    port: u16 = 8080,
};
```

A reader does not need to infer that `100` is a `u32` — it is stated explicitly.

---

## 8. Examples

### Example 1: Type-Safe Bitmask Operations

This example demonstrates arbitrary-width integers, bitwise operations with wrapping semantics, and the `@intCast` family in a realistic pattern — extracting fields from a packed binary value:

```zig
const std = @import("std");

// A hypothetical 16-bit network header field:
//   bits [15:12] = version (4 bits)
//   bits [11:4]  = ttl     (8 bits)
//   bits [3:0]   = flags   (4 bits)
const Header = packed struct(u16) {
    flags:   u4,
    ttl:     u8,
    version: u4,
};

pub fn main() void {
    const raw: u16 = 0x4F02; // version=4, ttl=0xF0=240, flags=0x2

    // @bitCast reinterprets the bit pattern as a Header without any
    // arithmetic conversion. The bit layout matches because of `packed struct`.
    const hdr: Header = @bitCast(raw);

    std.debug.print("version={d} ttl={d} flags={d}\n", .{
        hdr.version,
        hdr.ttl,
        hdr.flags,
    });
    // output: version=4 ttl=240 flags=2
}
```

**Build:** `zig build-exe header_parse.zig`

### Example 2: Comptime Constants Driving Runtime Behaviour

```zig
const std = @import("std");

// These constants are comptime-evaluated. The array size is determined at
// compile time; no runtime allocation occurs.
const CHUNK_SIZE: usize = 512;
const MAX_CHUNKS: usize = 8;
const BUFFER_SIZE: usize = CHUNK_SIZE * MAX_CHUNKS; // 4096, computed at comptime

// Stack-allocated buffer whose size is a comptime constant.
// The compiler knows the exact stack frame size at compile time.
pub fn process_chunks(data: []const u8) void {
    std.debug.assert(data.len <= BUFFER_SIZE);

    var work_buf: [BUFFER_SIZE]u8 = undefined;
    @memcpy(work_buf[0..data.len], data);

    // Process in CHUNK_SIZE blocks.
    var offset: usize = 0;
    while (offset < data.len) {
        const end = @min(offset + CHUNK_SIZE, data.len);
        process_chunk(work_buf[offset..end]);
        offset = end;
    }
}

fn process_chunk(chunk: []u8) void {
    for (chunk) |*byte| {
        byte.* ^= 0xFF; // invert all bits
    }
}

pub fn main() void {
    var payload = [_]u8{ 0x01, 0x02, 0x03, 0x04 };
    process_chunks(&payload);
    std.debug.print("{x}\n", .{std.fmt.fmtSliceHexLower(&payload)});
}
```

**Build:** `zig build-exe chunk_process.zig`

### Example 3: The Full Casting Toolkit

```zig
const std = @import("std");

pub fn main() void {
    // 1. @as: safe coercion (comptime-verified, lossless)
    const small: u8 = 200;
    const wide: u16 = @as(u16, small); // u8 -> u16 always safe
    _ = wide;

    // 2. @intCast: runtime-checked narrowing (panics in Debug on overflow)
    const big: i64 = 42;
    const narrow: i16 = @intCast(big); // safe: 42 fits in i16
    _ = narrow;

    // 3. @truncate: bit-truncation, no check
    const full: u16 = 0x01FF;
    const low: u8 = @truncate(full); // takes low 8 bits: 0xFF = 255
    std.debug.print("truncated: {d}\n", .{low}); // 255

    // 4. @bitCast: reinterpret bits (same size)
    const signed: i32 = -1;
    const unsigned: u32 = @bitCast(signed); // 0xFFFFFFFF = 4294967295
    std.debug.print("bitcast: {d}\n", .{unsigned}); // 4294967295

    // 5. @floatFromInt / @intFromFloat
    const count: i32 = 7;
    const rate: f64 = @floatFromInt(count); // 7.0
    const rounded: i32 = @intFromFloat(@round(rate * 1.5)); // 10 (rounds 10.5 → 11? no: 10.5 rounds to 11 but @round is nearest-even)
    std.debug.print("rate={d:.1} rounded={d}\n", .{ rate, rounded });
}
```

**Build:** `zig build-exe casting.zig`

### Example 4: Block Expressions Replacing Multi-Branch Logic

```zig
const std = @import("std");

const Tier = enum { free, standard, premium };

pub fn rate_limit(tier: Tier, hour: u8) u32 {
    // Block expression: each branch produces the rate limit value.
    // No need for a separate variable assignment above the switch.
    const requests_per_minute: u32 = switch (tier) {
        .free     => blk: {
            // Free tier: 10 RPM, but throttled to 5 after hour 20.
            if (hour >= 20) break :blk 5;
            break :blk 10;
        },
        .standard => 100,
        .premium  => 1000,
    };

    return requests_per_minute;
}

pub fn main() void {
    std.debug.print("free@14h:  {d} rpm\n", .{rate_limit(.free,     14)});
    std.debug.print("free@21h:  {d} rpm\n", .{rate_limit(.free,     21)});
    std.debug.print("premium:   {d} rpm\n", .{rate_limit(.premium,   9)});
}
```

**Build:** `zig build-exe rate_limit.zig`

---

## 9. Summary & Exercises

### Summary

Zig's binding and type system is built on five interlocking ideas:

**`var` and `const` are explicit mutability declarations.** There is no default. `const` produces an immutable binding and, when the binding holds a pointer, makes the pointed-to data immutable through that pointer. `const` with a comptime-known value produces zero runtime overhead.

**Type inference is local and deterministic.** `comptime_int` and `comptime_float` are not runtime types; they resolve to concrete types at each use site through peer type resolution. There is no constraint propagation across statements. The type is always knowable by reading left-to-right.

**No implicit numeric promotion or coercion between concrete types.** Every type boundary is explicit. The casting builtins — `@intCast`, `@truncate`, `@bitCast`, `@floatFromInt`, `@intFromFloat`, `@floatCast`, `@as` — each have precise semantics. Choosing the right one is a self-documenting act of intent.

**`undefined` is a debug trap, not a zero value.** In Debug builds, it poisons memory with `0xaa`. In release builds it grants the optimizer maximum freedom. Do not read from `undefined`-initialised memory; initialise all paths before use.

**Shadowing is forbidden.** New declarations in inner scopes cannot reuse names from outer scopes. This is a readability and correctness constraint that forces explicit renaming.

These five properties flow from the same source: Zig places **legibility and auditability above syntactic ergonomics**. The cost is more keystrokes at declaration time. The benefit is a codebase where every type boundary, every mutation, and every initialisation decision is visible to the reader without running a type checker in their head.

---

### Exercises

**Exercise 4-1: The Bit-Field Protocol Parser**

Design a `packed struct` that maps a 32-bit little-endian binary record from a hypothetical sensor protocol. The layout is: bits [31:28] message type (4-bit enum), bits [27:16] sequence number (12-bit unsigned), bits [15:0] payload CRC (16-bit unsigned). Write a function that accepts a `u32` and returns the parsed struct using `@bitCast`. Write a second function that reconstructs the `u32` from a struct instance. Add a `test` block that round-trips 10 different values and verifies identity. Consider: what happens if you flip the field order? What does Zig's `packed struct` layout guarantee about byte order?

**Exercise 4-2: The Safe Integer Narrowing Wrapper**

The standard `@intCast` panics in Debug mode on overflow but is silent in ReleaseFast. Write a function `safe_narrow(comptime T: type, value: anytype) error{Overflow}!T` that uses `std.math.cast` (or implements the range check manually) to return an error union instead of panicking. Write a version that takes a custom error type as a comptime parameter. Benchmark the two approaches (erroring vs. panicking) in a tight loop on a 64-bit to 8-bit narrowing operation. What does the assembly look like in ReleaseFast for each?

**Exercise 4-3: A Compile-Time Unit System**

Using only `const`, `comptime_int`, and comptime arithmetic, build a unit-conversion system for distances that encodes the unit in the type. Define types `Metres`, `Kilometres`, and `Miles` as `struct { value: f64 }`, then write `comptime` conversion functions between them. The goal is to produce a compile error if you accidentally add a `Metres` value to a `Miles` value without an explicit conversion. Extend the system to handle the velocity derived unit `MetresPerSecond`. Where does this approach break down compared to a full dimensional-analysis library built on `comptime` type parameters?
