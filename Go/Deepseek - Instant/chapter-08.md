# Chapter 8: Arrays vs. Slices

If you’re coming from languages like C, Java, or Python, you might assume that Go’s arrays work like every other array you’ve used. You’d be wrong. Go’s arrays are deliberately awkward, and slices are the flexible, powerful collection you’ll actually use. Understanding why Go made this split — and how slices really work under the hood — is your first step toward writing memory-efficient, predictable Go code.

---

## 1. Basic Usage

Let’s start with the mechanics. Arrays are fixed-size sequences, declared with a length that’s part of their **type**. Slices are dynamically-sized views into arrays (or other slices).

```go
package main

import "fmt"

func main() {
    // Arrays: length is part of the type
    var arr1 [3]int                     // zero-valued: [0, 0, 0]
    arr2 := [3]int{1, 2, 3}             // literal
    arr3 := [...]int{1, 2, 3}           // compiler counts elements
    
    // Slices: no length in type
    var slice1 []int                    // nil slice (more on this later)
    slice2 := []int{1, 2, 3}            // creates an underlying array
    slice3 := make([]int, 3, 5)         // len=3, cap=5
    
    // Indexing works the same for both
    arr2[0] = 42
    slice2[0] = 42
    
    // Slicing: [low:high] (half-open, exclusive high)
    sub := slice2[1:3]                  // elements [42, 3]
    
    // Append: slices only
    slice2 = append(slice2, 4, 5)       // returns new slice if capacity exceeded
    
    fmt.Println(arr1, arr2, arr3)
    fmt.Println(slice1, slice2, slice3, sub)
}
```

**Key syntax distinctions:**
- `[3]int` vs `[]int` — the presence of a length means array.
- `...` in array literal lets the compiler count for you, but the result is still a fixed-size array.
- `make([]T, len, cap)` is the only way to create a slice with a specific initial capacity.
- Append always returns a **new slice** value, even if it didn’t reallocate.

---

## 2. Under the Hood

The “Aha!” moment for slices comes when you realize they’re just lightweight descriptors (a `slice` header) pointing to a backing array. Here’s the runtime representation from the Go source (`runtime/slice.go`):

```go
type slice struct {
    array unsafe.Pointer  // pointer to the first element of the backing array
    len   int             // number of accessible elements
    cap   int             // total capacity of the backing array from this pointer
}
```

When you pass a slice to a function, you’re copying this 3-word struct (24 bytes on 64-bit platforms). You’re **not** copying the backing array.

**Array semantics:**
- Arrays are **values** — copying an array copies every element (deep copy).
- Passing an array to a function copies the entire array (O(n) cost).
- The size of an array type is fixed at compile time, so `[1000000]byte` is a valid type, but using it on the stack may cause overflow.

**Slice semantics:**
- Copying a slice copies the header, not the elements.
- Multiple slices can share the same backing array.
- Append checks `len == cap`; if true, it allocates a new larger backing array (following a growth policy), copies existing elements, and then adds the new ones.

**The growth policy (Go 1.21+):**
- For capacity < 256, it roughly doubles (`cap * 2`).
- For larger capacities, it grows by ~25% (to reduce memory waste).
- The exact formula is in `runtime.growslice` — it’s size-class aware to reduce fragmentation.

**Example showing sharing:**

```go
func main() {
    a := []int{1, 2, 3, 4, 5}
    b := a[1:4]           // len=3, cap=4 (from index 1 to end of backing array)
    b[0] = 99             // modifies a[1] as well
    
    fmt.Println(a)        // [1, 99, 3, 4, 5]
    fmt.Println(b)        // [99, 3, 4]
    
    // Now append to b — may or may not reallocate
    b = append(b, 100)    // still within capacity, writes to a[4] (5 → 100)
    fmt.Println(a)        // [1, 99, 3, 4, 100]
    
    b = append(b, 200)    // cap exceeded, new backing array allocated
    b[0] = 999            // no longer affects a
    fmt.Println(a)        // unchanged: [1, 99, 3, 4, 100]
}
```

This sharing behavior is the source of both power and many bugs.

---

## 3. Why This Design?

Go’s split between arrays and slices is intentional and reflects the “less is more” philosophy. The Go team could have provided only dynamic arrays (like Python’s `list` or Java’s `ArrayList`). Instead, they chose two distinct types for two distinct use cases.

**Why arrays exist at all:**
1. **Explicit value semantics** — sometimes you truly want a fixed-size, copyable chunk of memory. For example, cryptographic hashes (`[32]byte`), UUIDs (`[16]byte`), or fixed-size matrices.
2. **Stack allocation** — small arrays can live entirely on the stack, avoiding GC pressure entirely.
3. **Type safety** — `[3]int` and `[4]int` are different types, preventing accidental size mismatches at compile time.

**Why slices are the primary collection:**
- Arrays are rigid — their size is fixed at compile time. You can’t write a function that accepts an array of any length unless you use slices.
- Slices add indirection, enabling dynamic resizing, sub-slicing, and shared memory.
- The header + pointer design allows efficient passing (no copying of elements).

**Why not one unified type?** (e.g., C++ `std::vector` or Rust `Vec<T>`)
Go could have made arrays resizeable by default, but that would hide allocation costs. Go’s design forces you to acknowledge when you’re using a dynamic, heap-allocated structure (slice) vs. a fixed, potentially stack-allocated one (array). The compiler can optimize arrays aggressively because their size is known. Slices require runtime checks.

The “Aha!” realization: **Arrays are about value semantics; slices are about reference semantics.** Choose based on whether you want copy-on-assignment (array) or shared access (slice).

---

## 4. Competing Approaches

| Language | Primary Collection | Value/Reference | Resizable | Memory Location |
|----------|-------------------|----------------|-----------|------------------|
| **Go** | Slice | Reference (header copied) | Yes | Heap (backing array) |
| **Go** | Array | Value (deep copy) | No | Stack or heap |
| **C** | Raw pointer + length | Reference | Manual | Any |
| **C++** | `std::vector` | Value (deep copy) | Yes | Heap |
| **Rust** | `Vec<T>` | Ownership (move) | Yes | Heap |
| **Python** | `list` | Reference | Yes | Heap |
| **Java** | `ArrayList` | Reference | Yes | Heap |
| **JavaScript** | `Array` | Reference | Yes | Heap |

**Key comparisons:**

- **vs. C++ `std::vector`**: Copying a vector in C++ copies all elements (unless moved). Copying a Go slice copies only the header. This means Go can be more efficient when passing slices around, but you must be aware of sharing. C++ gives you fine-grained control via copy/move constructors; Go gives you one simple, predictable model.

- **vs. Rust `Vec<T>`**: Rust’s ownership model makes sharing explicit (either via borrows or cloning). Go’s slices default to shared mutable access — powerful but dangerous. Rust prevents data races at compile time; Go relies on runtime detection (`-race` flag) and developer discipline.

- **vs. Python `list`**: Python’s lists are also references to dynamic arrays, but Python always copies the reference (no “array value” equivalent). Go’s arrays give you an escape hatch for copy semantics. Also, Python’s list growth factor is slightly different (≈12.5% for large lists), while Go’s is more aggressive initially.

- **vs. Java `ArrayList`**: Java’s generics mean you can’t have `ArrayList<int>` without boxing (use `IntArrayList` from third-party libs). Go’s slices are unboxed — `[]int` stores raw ints. No wrapper objects, no GC pressure for elements.

**Where Go shines**: Unboxed elements + header-copy semantics gives you a combination of performance (contiguous memory) and ergonomics (pass-by-header). You can’t get that in Java without custom collections.

**Where Go pays a price**: Shared backing array can lead to surprising mutations. Rust eliminates this category of bug entirely. Go says: “we trust you to be careful.”

---

## 5. Common Mistakes

### Mistake 1: Assuming `append` never shares
```go
func main() {
    original := []int{1, 2, 3}
    wrong := append(original, 4)  // might reuse backing array if capacity allows
    original[0] = 99
    fmt.Println(wrong[0])          // Could be 99 or 1 — depends on capacity!
}
```
**Fix**: If you want a truly independent copy, use `copy`:
```go
    independent := make([]int, len(original))
    copy(independent, original)
    independent = append(independent, 4)
```

### Mistake 2: Storing a slice after reallocation
```go
func addOne(s []int) {
    s = append(s, 1)  // s is a copy of the header, reassigning it does nothing to caller
}
```
**Fix**: Return the new slice:
```go
func addOne(s []int) []int {
    return append(s, 1)
}
```
Or pass a pointer to the slice: `func addOne(s *[]int)`

### Mistake 3: Slicing a slice and expecting independent memory
```go
data := []byte("long string")
sub := data[1:3]   // sub shares memory with data
// Later, data is garbage, sub keeps entire backing array alive
```
**Fix**: When you need a small slice from a large one, copy it:
```go
sub := make([]byte, 2)
copy(sub, data[1:3])
```

### Mistake 4: Nil slice vs. empty slice confusion
```go
var a []int          // nil slice, len=0, cap=0, pointer=nil
b := []int{}         // empty slice, len=0, cap=0, pointer to a zero-sized array
c := make([]int, 0)  // empty slice, may have non-nil pointer
```
`append` works identically on all, but JSON serialization treats `null` vs `[]` differently. Use `var a []int` for “no elements” and `[]int{}` when you need a non-nil representation.

### Mistake 5: Range loop with index overwriting
```go
items := []Item{{Name: "a"}, {Name: "b"}}
for _, item := range items {
    item.Name = "modified"  // works on a copy of the element!
}
```
**Fix**: Use index:
```go
for i := range items {
    items[i].Name = "modified"
}
```

---

## 6. Performance Considerations

**Allocation cost:**
- `[N]T` on stack: O(1) no heap allocation, zero GC pressure. Great for small, fixed-size buffers.
- `make([]T, len, cap)`: one heap allocation (if cap > small-threshold). Subsequent appends cause reallocations with O(n) copy cost.

**Growth cost amortization:**
- Append’s exponential growth ensures amortized O(1) per element. Each element is copied ~log₂(n) times total.
- For large slices (> 1M elements), the 25% growth factor means ~5 copies over the lifetime of the slice (since 1.25¹⁰ ≈ 9.3x). Tune initial capacity with `make` to avoid reallocation entirely.

**Memory overhead:**
- Slice header: 24 bytes per slice variable (on 64-bit).
- Backing array overhead: 8–16 bytes per allocation (GC metadata). Many small slices can waste memory.
- Sub-slice retains entire parent backing array. If you create a 10-byte slice from a 1GB array, the GC can’t collect the 1GB until all slices referencing it are gone.

**Copy cost:**
- Copying an array of 1000 ints: 8000 bytes copy.
- Copying a slice header: 24 bytes copy, regardless of size.
- `copy(dst, src)` copies in word-sized chunks (platform dependent), efficient for large blocks.

**Compiler optimizations:**
- Small arrays (≤4 elements?) may be unrolled in loops.
- `len()` and `cap()` on slices are O(1) (reading from header).
- Range loops over slices produce a single copy of each element; avoid if elements are large structs (use `for i := range` instead).

**When to use arrays:**
- Fixed-size buffers on hot paths (e.g., `[64]byte` for stack-allocated I/O).
- Types that are naturally fixed-size: IP addresses (`[4]byte` or `[16]byte`), hashes, coordinates.

**When to pre-allocate slice capacity:**
```go
// Bad: 5 allocations (1 initial + 4 grows)
s := []int{}
for i := 0; i < 1000; i++ {
    s = append(s, i)
}

// Good: 1 allocation
s := make([]int, 0, 1000)
for i := 0; i < 1000; i++ {
    s = append(s, i)
}
```

---

## 7. Best Practices (The Idiomatic Way)

1. **Use slices, not arrays, for function parameters** — unless you explicitly want copy semantics.
   ```go
   // Good
   func Process(items []int) []int
   
   // Rarely what you want
   func Process(items [10]int) [10]int
   ```

2. **Prefer `var s []T` for nil slices** — it’s idiomatic and works with `append`, `len`, `range`.
   ```go
   var users []User   // nil slice
   users = append(users, newUser)  // just works
   ```

3. **Return slices, not pointers to slices** — the header is small and returning a pointer adds indirection.
   ```go
   func GetItems() []Item { ... }   // Good
   func GetItems() *[]Item { ... }  // Avoid
   ```

4. **When you need a fresh copy, use `copy` with a pre-allocated target**:
   ```go
   dest := make([]T, len(src))
   copy(dest, src)
   ```

5. **Check capacity before sharing** — document if a function modifies the slice’s backing array.

6. **Use `append` assignment pattern consistently**:
   ```go
   slice = append(slice, elem)   // Always assign back
   ```

7. **For large structs in slices, store pointers or use index loops**:
   ```go
   for i := range largeStructSlice {
       largeStructSlice[i].Field = newValue
   }
   ```

8. **Use `slog` with slices carefully** — logging huge slices can blow up output. Log length instead.

---

## 8. Examples

### Example 1: Generic slice reversal (works with any slice type using Go 1.21+)
```go
func ReverseSlice[T any](s []T) {
    for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
        s[i], s[j] = s[j], s[i]
    }
}
// Note: modifies in place, no allocation
```

### Example 2: Efficient filter with pre-allocation
```go
func FilterInts(in []int, keep func(int) bool) []int {
    // Pre-allocate with capacity guess (worst case = len(in))
    out := make([]int, 0, len(in))
    for _, v := range in {
        if keep(v) {
            out = append(out, v)
        }
    }
    return out
}
```

### Example 3: Copy-on-write slice wrapper (useful for shared data)
```go
type CowSlice struct {
    data []int
    dirty bool
}

func (c *CowSlice) Get(i int) int {
    return c.data[i]
}

func (c *CowSlice) Set(i int, val int) {
    if !c.dirty {
        // Copy before first write
        newData := make([]int, len(c.data))
        copy(newData, c.data)
        c.data = newData
        c.dirty = true
    }
    c.data[i] = val
}
```

### Example 4: Fixed-size ring buffer using array (no GC allocation)
```go
type RingBuffer struct {
    buf [1024]byte  // On stack if embedded in struct that's on stack
    head int
    tail int
    full bool
}

func (r *RingBuffer) Write(b byte) {
    r.buf[r.head] = b
    r.head = (r.head + 1) % len(r.buf)
    if r.full {
        r.tail = (r.tail + 1) % len(r.buf)
    }
    r.full = r.head == r.tail
}
```

---

## 9. Summary & Exercises

### Summary
- **Arrays** are values with fixed compile-time size. Copy copies all elements. Use for small, fixed-size data where stack allocation matters.
- **Slices** are headers (ptr, len, cap) referencing a backing array. Copy copies the header (24 bytes). Use for almost all dynamic collections.
- `append` may share or reallocate — always reassign the result.
- Sub-slices share memory with parent — beware of memory leaks.
- Pre-allocate capacity when the final size is known or can be estimated.
- Nil slices are idiomatic and work like empty slices for most operations, except JSON serialization.

### Exercises

**Exercise 1: Detect and fix slice sharing bug**
Given the following code, a production service logs user activities. The `UserActivity` struct holds a slice of recent events. Why does modifying an activity in one part of the program corrupt another? Fix it without removing the concurrency.

```go
type UserActivity struct {
    UserID  int
    Events  []string
}

func NewUserActivity(id int, events []string) *UserActivity {
    return &UserActivity{UserID: id, Events: events}
}

// Later in the code...
alice := NewUserActivity(1, []string{"login"})
bob := NewUserActivity(2, alice.Events) // Intent: start bob with same events
bob.Events[0] = "purchase"              // corrupts alice's events!
```

**Exercise 2: Implement a thread-safe, bounded slice pool**
Write a `BoundedSlicePool` that maintains a fixed number of reusable `[]byte` slices of a given capacity. It should:
- Provide `Get() []byte` that returns a slice with length 0 but capacity at least the pool’s target capacity.
- Provide `Put(s []byte)` that returns the slice to the pool if its capacity is within a tolerance (e.g., not grown beyond 2× target).
- Never block: if pool is empty, allocate a new slice. If full, discard.
- Handle the case where the returned slice has been `append`ed beyond original capacity — the pool should only reuse if `cap(s) <= maxCap`.

**Exercise 3: Optimize a log parser using slice pre-allocation**
You have a function that reads a large log file (millions of lines) line by line, parses each line into a `struct { timestamp time.Time, level string, message string }`, and returns a slice of all parsed entries. The current version uses `append` without pre-allocation, causing many reallocations and high GC pressure. Rewrite it to:
- Perform one initial scan over the file to count lines (without parsing fully) or estimate capacity.
- Pre-allocate the slice with the exact size (or a good estimate).
- Return the slice of parsed entries.
- Benchmark the difference (conceptually — you don’t need to run benchmarks, but describe the expected improvement and any trade-offs).

**Expected learnings:**
- Ex1: Understanding reference sharing via slices.
- Ex2: Combining slices with synchronization and capacity checks.
- Ex3: Real-world optimization using capacity hints, and the trade-off of double-scanning the file (I/O vs allocation reduction).
