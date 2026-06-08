## Chapter 8: Arrays vs. Slices

Go offers two primary ordered collection types: arrays and slices. You’ll see arrays in struct fields or low-level buffer declarations; you’ll see slices everywhere else. Understanding the exact memory semantics of each – and how slices build on arrays – is essential to writing correct, allocation-efficient Go. This chapter unpacks the array’s value semantics, the slice’s lightweight header, the `append` mechanics that trip up even experienced engineers, and the trade-offs that make slices Go’s workhorse data structure.

### 1. Basic Usage

#### Arrays

An array’s size is part of its type, fixed at compile time. Arrays are **value types**; assigning or passing an array copies all its elements.

```go
// A fixed-length array of 4 bytes; zero value is [0,0,0,0].
var buffer [4]byte

// Array literal with explicit size.
primes := [3]int{2, 3, 5}

// Use ... to infer size from literal.
weekdays := [...]string{"Mon", "Tue", "Wed", "Thu", "Fri"}

// Access and iteration.
buffer[0] = 0xFF
for i, v := range primes {
    fmt.Println(i, v)
}
```

Passing an array to a function copies it:

```go
func zero(buf [4]byte) {
    for i := range buf {
        buf[i] = 0
    }
}

var b [4]byte
b[0] = 42
zero(b)
fmt.Println(b) // [42 0 0 0] – original unchanged
```

You rarely want this copy cost for anything beyond tiny arrays. For larger fixed-size groups, you’ll typically pass a pointer to an array or, more idiomatically, a slice.

#### Slices

A slice is a **descriptor** (header) that references a contiguous segment of an underlying array. The `[]T` type does not include a length.

```go
// Create a slice with make: type, length, (optional) capacity.
s := make([]int, 3, 5)      // len=3, cap=5; backing array of 5 ints zeroed.

// Literal creates a slice backed by an anonymous array.
names := []string{"Alice", "Bob"}

// Nil slice: zero value; no backing array.
var empty []int              // nil, len=0, cap=0

// Slice an existing array or slice.
arr := [5]int{10, 20, 30, 40, 50}
sub := arr[1:4]              // []int{20,30,40}, len=3, cap=4 (from index 1 to end of arr)
```

The built-in functions `len`, `cap`, `append`, and `copy` are the slice toolkit:

```go
s := make([]int, 0, 4)
s = append(s, 1)           // len=1 cap=4
s = append(s, 2, 3, 4)     // len=4 cap=4
s = append(s, 5)           // len=5 cap=8 (grew)

// Copy copies min(len(dst), len(src)) elements; returns count.
dst := make([]int, 3)
n := copy(dst, s)           // n = 3, dst = [1,2,3]
```

Slicing creates a **new header** pointing into the **same** backing array:

```go
a := []int{1, 2, 3, 4, 5}
b := a[2:4]    // [3,4], len=2, cap=3 (from index 2 to end of a's array)
b[0] = 99
fmt.Println(a) // [1, 2, 99, 4, 5]
```

### 2. Under the Hood

The slice header is a small struct with three fields:

```go
type slice struct {
    ptr unsafe.Pointer  // pointer to first element of the segment
    len int
    cap int
}
```

When you assign a slice to another variable, you **copy the header**, not the data. The `ptr` field remains the same, so both slices share the underlying array. This is the source of both power and bugs.

An array, by contrast, has no indirection; the entire element storage is laid out inline. Copying an array copies all bytes.

#### `append` Mechanics

When you call `append`, the runtime checks whether `len < cap`:

- **Enough capacity**: The element is placed at index `len`, `len` is incremented by one, and the **same backing array** is returned (via a new slice header with the updated length). Any other slices that share this backing array now see the mutation.
- **Insufficient capacity**: A new, larger backing array is allocated. The old elements are copied into it, the new element(s) are appended, and a new slice header pointing to the fresh array is returned. The original slice(s) still reference the old array, which remains unchanged and will be garbage collected when no longer referenced.

The growth formula is part of the `growslice` routine in `runtime/slice.go`:

- If the desired capacity is more than double the current capacity, it jumps straight to that capacity.
- Otherwise, if the old capacity is less than 256, the new capacity is **double** the old.
- For capacities ≥ 256, it increases by approximately 1.25× per iteration, smoothing out the transition using a formula that adds `(oldcap + 3*256) / 4`.

This amortises allocation cost to O(1) per append, but every growth incurs a memory allocation and an element copy – something to be mindful of in hot paths.

#### Memory and Escape Analysis

The slice header itself lives on the stack (or in a register), but the **backing array** may be heap‑allocated. The compiler’s escape analysis decides:

- `make([]T, …)` typically allocates the backing array on the heap because the resulting pointer escapes to the caller or persists beyond the function.
- If the compiler can prove the slice does not escape (e.g., a small temporary buffer used only inside a function), it might allocate the backing array on the stack. This is a powerful optimisation for short‑lived, fixed‑size slices.
- Taking a slice of a stack‑allocated array (`arr[:]`) keeps the backing array on the stack **if** the array itself doesn’t escape – but slicing often exposes a pointer, which can force the array to heap.

Understanding escape analysis helps you avoid surprising heap allocations; you can inspect it with `go build -gcflags="-m"`.

### 3. Why This Design?

Why does Go have **both** arrays and slices, and why do arrays have copy‑by‑value semantics?

**Arrays as value types** make data ownership explicit and simple. When you pass an array, you know the callee cannot mutate the caller’s data. This eliminates a whole class of aliasing bugs. The same reasoning underlies Go’s default pass‑by‑value for structs. It’s a conscious choice that prioritises **local reasoning** over the convenience of reference semantics.

However, copying large arrays is expensive, and fixed‑size collections don’t fit most dynamic needs. The slice is Go’s answer: a flexible, safe reference type that still gives the programmer control over memory sharing. By making the slice a lightweight header, the language decouples the view from the storage. You can pass a slice to a function cheaply (just the 24‑byte header) while still being able to mutate the underlying data if desired.

The design pushes you toward slices as the **primary collection**:
- Ranging over an array copies the whole array before iteration, a subtle performance trap.
- Arrays can’t grow; you’d need to manually allocate, copy, and track length.
- Arrays are rarely used as function parameters or return types in idiomatic Go; you almost always see `[]byte`, `[]int`, etc.

Why didn’t Go simply make arrays reference types, like Java or Python? Because then every array assignment would introduce aliasing, making it harder to reason about state. The slice makes the sharing **explicit** – you can see by the `[]T` type that you’re dealing with a view, and you decide whether to `copy` or `append` (which may or may not isolate) based on your needs.

Finally, the `append` behaviour – sometimes mutating in place, sometimes allocating a new array – is intentional. It gives you the performance of in‑place modification when possible, while still guaranteeing that the slice never overflows. You’re expected to always use the returned slice (`s = append(s, v)`) and never depend on the old header.

### 4. Competing Approaches

**Python / JavaScript**: Both treat lists/arrays as reference types with dynamic resizing. A Python list `l` is a pointer to a heap‑allocated structure; assigning `l2 = l` creates an alias, not a copy. Go slices behave similarly in aliasing, but with the crucial difference that `append` may break aliasing silently. Python’s `list.append` always mutates in place. JavaScript `Array` methods like `concat` return new arrays, but assignment is shared. Go gives you finer‑grained control: you choose between sharing (slicing, direct assignment) and isolation (`copy`, or `append` that causes a capacity change).

**Java / C#**: `ArrayList<T>` is a class with reference semantics. Passing an `ArrayList` passes the reference, so mutation is shared. Java arrays are also reference objects; `int[] a = b` aliases. Go’s array value semantics prevent that, forcing you to explicitly opt‑in to sharing via slices. The Java approach makes it easy to mutate data unexpectedly across module boundaries; Go’s approach forces you to either pass a pointer to the array (explicit) or a slice (clear).

**C++**: `std::vector<T>` is a class with value semantics: copying a vector copies all elements (deep copy), similar to Go’s array but with dynamic size. C++20’s `std::span` is a non‑owning view that closely resembles a Go slice header. However, in C++ you can also work with raw arrays and pointers, and the language doesn’t prevent dangling references. Go’s GC makes slices safe; a slice always points to valid memory (or nil) because the backing array lives as long as any referencing slice. The trade‑off is GC overhead vs. C++’s deterministic ownership model (unique_ptr, shared_ptr). C++ gives you more control over allocation; Go abstracts that away.

**Rust**: Rust’s type system draws the sharpest analogue. A Rust array `[T; N]` is a value type on the stack; a slice `&[T]` is a borrowed view (a fat pointer: `*const T` + `len`). `Vec<T>` is an owning, growable heap allocation. Go’s slice is like a fusion of `Vec<T>` and `&[T]` – it can own or borrow depending on context, with the GC taking care of lifetimes. The key difference is ownership tracking: Rust’s borrow checker prevents data races and dangling slices at compile time; Go relies on runtime safety (bounds checking) and the GC. Rust forces you to think about lifetimes; Go lets you share memory freely, with the understanding that `append` might alias or not.

### 5. Common Mistakes

#### The Slice Append Trap

The most insidious bug in Go: passing a slice to a function that appends to it, expecting the caller to see the new elements. Since `append` may reallocate and change the `ptr`, the caller’s original header remains pointing at the old (often too‑small) array.

```go
func addItem(items []int, v int) {
    items = append(items, v) // if cap insufficient, new backing array
    fmt.Println("inside:", items)
}

func main() {
    s := make([]int, 0, 2)
    s = append(s, 10)
    addItem(s, 20)
    fmt.Println("outside:", s) // still [10], because addItem's append didn't change main's header
}
```

The fix: return the new slice, or pass a pointer to the slice.

#### Unintended Mutation Through Sharing

Slicing an existing slice creates a new header that shares the same backing array. Modifying elements through one view affects all others. This can be intentional, but it’s a common source of bugs when slices are passed deep into a call chain.

```go
original := []int{1, 2, 3, 4, 5}
window := original[1:3] // [2,3], cap=4
window[0] = 99          // original becomes [1,99,3,4,5]
```

Even trickier: `append` to a slice that still has capacity will modify the overlapping area of other slices.

#### Subslice Pinning a Large Array

When you take a small subslice from a large backing array, the subslice’s `ptr` still points into that large array. Even if the original large slice goes out of scope, the backing array **cannot be garbage collected** as long as any subslice references any part of it.

```go
func loadBig() []byte {
    huge := make([]byte, 10<<20) // 10 MB
    // … fill with data
    return huge[:16]             // return just 16 bytes, but entire 10MB stays alive
}
```

The correct approach is to copy the needed portion: `result := make([]byte, 16); copy(result, huge[:16])`.

#### Range Variable Capture

In `for _, v := range slice`, `v` is a **copy** of the element. If you take its address, you get the same stack address every iteration – a classic goroutine bug that also applies to slices.

```go
for _, v := range items {
    go func() {
        process(&v) // all goroutines process the last element
    }()
}
// Fix: assign to a local variable inside the loop or pass as argument.
```

#### Nil vs. Empty Slice

A nil slice (`var s []int`) and an empty slice (`s := []int{}` or `make([]int, 0)`) both have length 0, but their JSON representation differs: `nil` → `null`, empty → `[]`. In APIs that distinguish between “not set” and “empty set”, this matters. Functionally, you can `append` to both, and `len` returns 0.

### 6. Performance Considerations

#### Big‑O Costs

- **Indexing**: O(1) for both arrays and slices (bounds‑checked, but that’s a few CPU instructions).
- **Slicing**: O(1) – only a header copy, no allocation.
- **Copy**: O(min(len(dst), len(src))). Copies bytes directly, highly optimised.
- **Append**: Amortised O(1) per element, but each capacity growth involves O(n) copying of existing elements. This is why preallocation matters.

#### Allocation & GC Pressure

Every capacity‑doubling creates a new heap allocation and abandons the old one, which the GC must eventually collect. In a tight loop, this can cause significant GC pauses. **Preallocate with `make([]T, 0, expectedSize)`** whenever you know the approximate final length.

```go
// Good: one allocation for the backing array.
s := make([]string, 0, 1000)
for i := 0; i < 1000; i++ {
    s = append(s, strconv.Itoa(i))
}
```

#### Copy vs. Append

When you need to duplicate a slice, `copy` is generally more efficient than `append` because it avoids an extra allocation check if the destination already has enough capacity. If you need to duplicate and possibly extend, `append` is convenient but may allocate.

#### Subslice Memory Retention

As mentioned in “Common Mistakes”, holding a small subslice of a huge backing array wastes memory and increases GC work. The GC traces the entire array because the subslice’s pointer references it. In memory‑sensitive long‑lived caches, always copy the data you need.

#### Stack vs. Heap

Arrays and small slices can stay on the stack if escape analysis permits. For instance, a fixed‑size buffer `var buf [64]byte` never escapes if only used within a function. Slices created from such arrays with `buf[:]` also keep the data on the stack. This can eliminate heap allocation entirely for temporary buffers, a crucial technique in high‑performance networking code (e.g., `io.CopyBuffer`).

#### Using `clear` (Go 1.21+)

The built‑in `clear(s)` zeroes all elements of a slice without changing its length or capacity. This is a cheap way to reset a slice for reuse without deallocating and reallocating the backing array.

```go
buf := make([]byte, 0, 4096)
for {
    // read into buf after growing as needed
    process(buf)
    clear(buf)         // zero the contents
    buf = buf[:0]      // reset length but keep capacity
}
```

### 7. Best Practices

1. **Use slices for almost everything.** Arrays are for very specific low‑level needs: constants, checksums, small embedded buffers. Public API signatures should use slices.

2. **Preallocate with `make` when you know the size.** It’s one of the simplest and most impactful optimisations in Go.

3. **Always use the returned slice from `append`.** Never ignore it, never depend on the old slice having the new data. Idiomatic form: `s = append(s, v)`.

4. **Copy to isolate.** If you receive a slice from an untrusted caller or from a cache, and you need to prevent mutation, copy it: `safe := make([]T, len(src)); copy(safe, src)`.

5. **Return slices, not arrays or pointers to arrays.** Returning a slice is cheap (24 bytes) and idiomatic. If you must return a fixed number of values, consider a struct with a slice field or multiple return values.

6. **When slicing into a large array, copy if you need to keep a small part.** This prevents memory leaks.

7. **Prefer nil slices for “no data”.** Use `var s []T` to declare a nil slice. It’s idiomatic for “empty but not ready” and serialises to `null`. Use `make([]T, 0)` when you need an empty JSON array.

8. **Use `copy` to shift elements instead of manually looping.** For example, removing an element from a slice can be done with `copy(s[i:], s[i+1:])` followed by `s = s[:len(s)-1]`. This preserves capacity and avoids allocations.

9. **Consider `slog` and context where appropriate (not slice‑specific, but a general Go 1.21+ reminder).** The `slog` package uses slices of `any` for attributes; preallocating attribute slices improves log throughput.

10. **Leverage `clear` to reuse slices.** Reset to zero length and clear the data without releasing the underlying array, reducing GC churn.

### 8. Examples

#### Example 1: Array Value Semantics vs. Slice Sharing

```go
arr := [3]int{1, 2, 3}
arr2 := arr       // full copy
arr2[0] = 99
fmt.Println(arr)  // [1 2 3]

s := []int{1, 2, 3}
s2 := s           // copy of header, shares backing array
s2[0] = 99
fmt.Println(s)    // [99 2 3]
```

#### Example 2: Append and Capacity Demonstrating Sharing Break

```go
a := make([]int, 2, 4)
a[0], a[1] = 1, 2
b := a            // b shares backing array
a = append(a, 3)  // len 3, cap 4 – still same array
b = append(b, 4)  // len 3, cap 4 – overwrites a's [2]
fmt.Println(a)    // [1 2 4]
fmt.Println(b)    // [1 2 4]

// Now force a to grow beyond capacity:
a = append(a, 5, 6) // new backing array
b[0] = 99
fmt.Println(a)    // [1 2 4 5 6]
fmt.Println(b)    // [99 2 4] – different arrays now
```

#### Example 3: Preventing Subslice Memory Retention

```go
func readHeader(conn net.Conn) ([]byte, error) {
    buf := make([]byte, 4096)
    n, err := conn.Read(buf)
    if err != nil {
        return nil, err
    }
    // BAD: return buf[:n] // keeps 4KB alive
    // GOOD: copy only the needed bytes
    header := make([]byte, n)
    copy(header, buf[:n])
    return header, nil
}
```

#### Example 4: Using `copy` to Delete an Element Without Allocation

```go
func removeAt(s []int, i int) []int {
    copy(s[i:], s[i+1:])
    return s[:len(s)-1]
}
```

### 9. Summary & Exercises

**Summary**

- Arrays are fixed‑size value types; assignment copies all elements. They are rarely used directly in Go code except for small, compile‑time‑known buffers.
- Slices are lightweight view descriptors (pointer, length, capacity) that reference an underlying array. They are the primary collection type.
- Slicing creates a new header but shares the backing array, making mutation visible across slices.
- `append` grows the slice in place if capacity permits; otherwise it allocates a new, larger array and copies elements. This dual behaviour requires always capturing the returned slice.
- Common pitfalls include unintended mutation through sharing, subslice memory pinning, and the append trap.
- Performance gains come from preallocation, copying to isolate, and reusing backing arrays via `clear` and length resets.
- Go’s design favours slices because they combine safe sharing with flexible growth, while arrays stay true to value semantics for clear local reasoning.

**Exercises**

1. **Thread‑Safe Ring Buffer**: Implement a generic ring buffer (circular buffer) `RingBuffer[T]` backed by a slice with a fixed capacity. It must support `Push(T) bool` (returns false if full), `Pop() (T, bool)`, and `Len() int`. Ensure it is safe for concurrent use by a single producer and a single consumer without mutexes (hint: use atomics to track head/tail indices, but be mindful of memory ordering; Go’s atomic package can help). Provide a benchmark comparing it to a buffered channel of the same capacity.

2. **Slice Lifecycle Inspector**: Write a function `inspectAppend(s []int, vals ...int) (newSlice []int, reallocated bool)` that appends `vals` to `s` and returns whether a new backing array was allocated. You can determine this by capturing the original slice’s `unsafe.Pointer` to the first element and comparing it after the append. Use this to demonstrate the capacity doubling behaviour with a series of prints. (Note: `unsafe` is for learning purposes here; in production, avoid it.)

3. **Fix the Memory Leak**: The following code caches headers from a large file parser. Identify and fix the memory retention bug. Then write a unit test that, using `runtime.ReadMemStats`, verifies that the large underlying array is no longer referenced after the fix.

```go
type Parser struct {
    cache map[string][]byte
}

func (p *Parser) Parse(data []byte) {
    // data is typically 100MB+
    for _, line := range lines(data) {
        key, headerBytes := extractHeader(line)
        // BUG: headerBytes is a subslice of line, which points into data.
        p.cache[key] = headerBytes
    }
}
```
