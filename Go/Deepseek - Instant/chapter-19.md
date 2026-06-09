## Chapter 19: Pointers & Memory Allocation

### 1. Basic Usage

Pointers in Go hold the memory address of a variable. The syntax is deliberately minimal: `*T` denotes a pointer to a `T`, `&` takes the address, and `*` dereferences.

```go
package main

import "fmt"

func main() {
    // Basic pointer operations
    var x int = 42
    var p *int = &x      // p points to x
    fmt.Println(*p)      // 42 - dereference
    *p = 100             // modify x through p
    fmt.Println(x)       // 100

    // Pointer to struct
    type Point struct{ X, Y int }
    pt := Point{10, 20}
    pp := &pt
    // No need for explicit dereference: Go automatically handles it
    pp.X = 30            // Equivalent to (*pp).X = 30
    fmt.Println(pt)      // {30, 20}

    // Zero value of a pointer is nil
    var nilPtr *int
    if nilPtr == nil {
        fmt.Println("nil pointer")
    }
    // Dereferencing nil panics at runtime
    // *nilPtr = 5 // panic: runtime error: invalid memory address

    // new() vs literal
    p2 := new(int)       // allocates zeroed int, returns *int
    *p2 = 99
    p3 := &int{}         // same effect, more idiomatic for non-zero init
}
```

**Pointer receivers on methods** are the primary mechanism for mutation:

```go
type Counter struct{ val int }

func (c *Counter) Inc() { c.val++ }      // pointer receiver
func (c Counter) Value() int { return c.val } // value receiver

func main() {
    c := Counter{}
    c.Inc()              // Go automatically takes address: (&c).Inc()
    fmt.Println(c.Value()) // 1
}
```

### 2. Under the Hood

#### Memory Layout

A pointer in Go is a machine word (8 bytes on 64-bit architectures) containing a virtual memory address. The compiler and runtime track which pointers point to which heap objects for garbage collection.

Unlike C, Go pointers **cannot participate in arithmetic** (`p++` is illegal). This eliminates an entire class of memory safety bugs while retaining the ability to share references.

#### Stack vs. Heap Allocation

The Go compiler performs **escape analysis** to decide where a variable lives:

- **Stack**: Automatic, zero-cost allocation. The variable is tied to the function call frame. No GC pressure.
- **Heap**: Dynamic allocation. The garbage collector must scan and eventually free the memory.

```go
func stackAlloc() int {
    x := 42        // x lives on stack
    return x       // value returned, no pointer escapes
}

func heapAlloc() *int {
    x := 42        // x appears to be local...
    return &x      // BUT address escapes to caller -> heap allocation
}

func main() {
    p := heapAlloc() // p points to heap-allocated int
    _ = p
}
```

Compile with `go build -gcflags="-m"` to see escape analysis decisions:

```bash
$ go build -gcflags="-m" main.go
# command-line-arguments
./main.go:9:2: moved to heap: x   # because &x escapes
./main.go:13:13: ... argument does not escape
```

#### Escape Analysis Rules (Simplified)

The compiler aggressively promotes to heap when:

1. **Address taken and returned** (as above).
2. **Address stored in a global variable or interface** that outlives the function.
3. **Closure captures a variable by reference**.
4. **Value is too large for stack** (stack frame limits – typically 1KB per frame, but variable-size frames exist).
5. **Type contains pointers and is used in a way that confuses the compiler** (rare).

Conversely, the compiler *may* stack-allocate even when you take an address, if it can prove the pointer does not escape.

```go
func noEscape() {
    var buf [1024]byte
    p := &buf[0]      // Address taken, but p never leaves this function
    _ = p
    // buf allocated on stack because escape analysis proves safety
}
```

### 3. Why This Design?

Go’s pointer design reflects its philosophy of **controllable efficiency with safety**.

#### Why Pointers at All? Why Not Just Java-Style References?

Java references are opaque handles that are *always* pointers under the hood, but you cannot explicitly dereference or take addresses of local variables. Go gives you explicit control because:

- **Performance predictability**: You know when you’re passing a reference vs. a value. No hidden heap allocations.
- **Direct memory layout**: Pointers allow building efficient data structures (linked lists, trees, arenas) without runtime overhead.
- **Zero-cost abstractions**: A pointer-to-struct is just a memory address – no header word, no vtable (unless interface is involved).

#### Why No Pointer Arithmetic?

C’s `ptr++` is the source of countless buffer overflows, use-after-free, and type-punning bugs. Go’s designers chose **safety over flexibility** in this dimension. If you truly need pointer arithmetic (e.g., for serialization or high-performance parsing), you can use the `unsafe` package – but you explicitly sign away safety guarantees.

```go
import "unsafe"

// Legal but dangerous – opt-in unsafety
arr := [4]int{1,2,3,4}
ptr := unsafe.Pointer(&arr[0])
next := unsafe.Add(ptr, unsafe.Sizeof(int(0))) // arithmetic
value := *(*int)(next)
fmt.Println(value) // 2
```

#### Why Not Value-Only (Like Early Pascal)?

Pure value semantics force copying for every parameter or mutation, which is either slow (large structs) or forces manual heap management via allocation functions. Pointers give the best of both worlds: shared access without copy overhead, but with compile-time restrictions.

### 4. Competing Approaches

| Language | Pointer / Reference Model | Safety | Flexibility |
|----------|--------------------------|--------|-------------|
| **Go** | Explicit pointers, no arithmetic, GC | High – nil dereference panics, no dangling pointers | Medium – can’t do pointer math without `unsafe` |
| **Java** | Implicit references for objects, no pointer syntax | High – always safe, but hidden heap allocations | Low – no control over memory layout |
| **C++** | Raw pointers, references, smart pointers; arithmetic allowed | Low – use-after-free, buffer overflows common | Very high – full control, but easy to mis-use |
| **Rust** | References with lifetimes, raw pointers (unsafe), no GC | Very high – borrow checker enforces memory safety at compile time | High – safe pointer arithmetic via slices and iterators |
| **Python** | Everything is a reference, no pointer syntax | Very high – runtime checks | Very low – no direct memory access |
| **Zig** | Explicit pointers, optional pointer arithmetic, manual memory | Medium – safety checks in debug mode | Very high – full control, no hidden control flow |

**Key Go advantage**: You get the performance of explicit pointers without the mental overhead of Rust’s lifetimes or the danger of C++’s raw pointers. The garbage collector removes use-after-free entirely, and nil pointer dereferences are the only remaining hazard – and they panic predictably.

**Trade-off**: Go trades compile-time lifetime guarantees (Rust) for runtime simplicity. For most systems services, this is acceptable.

### 5. Common Mistakes

#### Mistake 1: Unnecessary Pointer to Primitive

```go
// Bad: heap allocates a bool for no reason
func isEnabled(flag *bool) bool {
    return *flag
}

// Good: pass value (bool fits in register, no allocation)
func isEnabled(flag bool) bool {
    return flag
}
```

**Why it matters**: `bool`, `int`, `float64`, and small structs (≤ machine word) are cheaper to copy than to indirect via pointer. A pointer dereference adds a memory load and prevents CPU register optimization.

#### Mistake 2: Returning Pointers to Loop Iteration Variables

```go
func makePointers() []*int {
    var result []*int
    for i := 0; i < 3; i++ {
        result = append(result, &i) // All pointers point to the SAME i
    }
    return result
}
// All three pointers dereference to 3 (the final value of i)
```

**Correct approach**: Create a new variable inside the loop or use a slice of values.

```go
func makePointers() []*int {
    var result []*int
    for i := 0; i < 3; i++ {
        val := i        // new variable each iteration
        result = append(result, &val)
    }
    return result
}
```

#### Mistake 3: Storing Pointers to Slice Elements After Append

```go
s := make([]int, 1, 1)
s[0] = 42
p := &s[0]      // p points to underlying array element
s = append(s, 99) // may allocate new array, move data
fmt.Println(*p)   // BUG: p may now point to old, freed memory (GC hasn't collected)
// In practice, this is a dangling pointer – but Go's GC won't collect the old array until p is unreachable.
// However, p now points to a different slice header's obsolete backing array.
```

**Safe pattern**: Avoid storing pointers to slice elements if the slice may grow. Use indices instead.

#### Mistake 4: Mixing Value and Pointer Receivers Inconsistently

```go
type Cache struct{ data map[string]string }

func (c Cache) Get(k string) string { return c.data[k] }      // value receiver
func (c *Cache) Set(k, v string)   { c.data[k] = v }         // pointer receiver

func main() {
    c := Cache{data: make(map[string]string)}
    c.Set("a", "b")   // fine: Go automatically takes &c
    // But the method set of value type Cache does NOT include Set
    // So this fails:
    // var c2 Cache = GetCache()
    // c2.Set("a", "b") // compile error: c2.Set undefined (value type)
}
```

**Rule**: If any method needs a pointer receiver (because it mutates), *all* methods on that type should use pointer receivers for consistency. Otherwise, users will be surprised when a value copy doesn’t implement the intended interface.

#### Mistake 5: Nil Pointer Dereference in Hot Paths

```go
func process(conf *Config) {
    // Assume conf is never nil... but what if it is?
    timeout := conf.Timeout  // PANIC if conf == nil
}
```

**Idiomatic guard clause**:

```go
func process(conf *Config) {
    if conf == nil {
        return // or return zero value, or error
    }
    timeout := conf.Timeout
}
```

### 6. Performance Considerations

#### Allocation Cost Breakdown

| Operation | Cost (ns) | GC Impact |
|-----------|-----------|-----------|
| Stack allocation (local var) | ~1 ns | None |
| Heap allocation (small object) | ~50-100 ns | Adds to GC scan work |
| Heap allocation (large object >32KB) | ~500 ns | May be allocated directly from OS (no per-object GC overhead) |
| Pointer dereference | ~1-3 ns (L1 cache hit) | None |

#### Benchmark: Pointer vs. Value

```go
type LargeStruct struct {
    Data [128]byte // 128 bytes
}

func ValueParam(s LargeStruct) LargeStruct { return s }
func PointerParam(s *LargeStruct) *LargeStruct { return s }

func BenchmarkValue(b *testing.B) {
    s := LargeStruct{}
    for i := 0; i < b.N; i++ {
        _ = ValueParam(s)
    }
}

func BenchmarkPointer(b *testing.B) {
    s := &LargeStruct{}
    for i := 0; i < b.N; i++ {
        _ = PointerParam(s)
    }
}

// Typical results:
// BenchmarkValue-8      200000000    7.8 ns/op    0 allocs/op
// BenchmarkPointer-8    1000000000   0.78 ns/op   0 allocs/op
```

For a 128-byte struct, pointer passing is ~10x faster because it copies only 8 bytes instead of 128. However, for a 8-byte struct (e.g., `int64`), value passing is actually faster because it avoids an extra dereference.

#### GC Pressure Escalation

Each heap-allocated pointer adds work for the garbage collector:

1. **Mark phase**: GC must traverse every pointer on the heap. More pointers → more scanning.
2. **Write barrier**: Assignments to pointers (e.g., `p.next = q`) trigger a write barrier, adding overhead.
3. **Pacing**: Frequent heap allocations cause the GC to run more often, stealing CPU time.

**Practical rule**: Keep hot-path allocations on the stack. If you must allocate, batch allocations (e.g., `make([]T, 0, preallocSize)`).

### 7. Best Practices

#### 1. Prefer Values for Small, Immutable Types

```go
// Good: time.Time is a small struct (24 bytes) with value semantics
func FormatTime(t time.Time) string {
    return t.Format(time.RFC3339)
}

// Bad: unnecessary pointer
func FormatTime(t *time.Time) string {
    return t.Format(time.RFC3339) // still dereferences
}
```

#### 2. Use Pointers for Large Structs (>64 bytes) or Required Mutability

```go
type BigBuffer struct {
    data [8192]byte
    pos  int
}

// Good: pointer receiver for mutation and size
func (b *BigBuffer) Write(p []byte) (n int, err error) {
    copy(b.data[b.pos:], p)
    b.pos += len(p)
    return len(p), nil
}
```

#### 3. Return Pointers Only When Necessary

```go
// Good: returns value by default
func NewConfig() Config { return Config{Timeout: 30} }

// Only return pointer if the zero value is meaningful as nil
func NewOptionalConfig() *Config {
    if disabled {
        return nil  // nil means "use defaults"
    }
    return &Config{Timeout: 30}
}
```

#### 4. Prefer `&T{}` over `new(T)`

```go
p := &MyStruct{Field: "value"} // idiomatic, initializes fields
// vs
p2 := new(MyStruct) // returns zeroed struct, rarely what you want
```

#### 5. Use `sync.Pool` for Frequently Allocated Pointers

```go
var bufferPool = sync.Pool{
    New: func() interface{} { return make([]byte, 4096) },
}

func process() {
    buf := bufferPool.Get().([]byte)
    defer bufferPool.Put(buf) // return to pool
    // use buf...
}
```

This reduces GC pressure for short-lived, repeatedly allocated objects.

#### 6. Document Pointer Behavior in APIs

```go
// GetUser returns a pointer to a User. It returns nil if the user does not exist.
func GetUser(id string) *User { ... }

// Update modifies u in place. u must not be nil.
func (db *DB) Update(u *User) error { ... }
```

### 8. Examples

#### Example 1: Escape Analysis Observation

```go
package main

func main() {
    // Case 1: does NOT escape
    var stackBuf [1024]byte
    _ = stackBuf

    // Case 2: escapes to heap because it's returned
    heapPtr := heapAlloc()
    _ = heapPtr

    // Case 3: escapes because it's passed to fmt.Println (interface{})
    x := 42
    fmt.Println(x) // forces heap allocation for x
}

func heapAlloc() *int {
    y := 100
    return &y // escapes to heap
}
```

Run with: `go build -gcflags="-m -l" main.go`

#### Example 2: Linked List with Pointers

```go
package main

import "fmt"

type Node struct {
    Value int
    Next  *Node
}

// Append adds a new node at the end (pointer receiver to modify head)
func (head *Node) Append(val int) *Node {
    newNode := &Node{Value: val}
    if head == nil {
        return newNode
    }
    current := head
    for current.Next != nil {
        current = current.Next
    }
    current.Next = newNode
    return head
}

// Walk traverses the list (value receiver is fine - no mutation)
func (head Node) Walk() {
    for current := &head; current != nil; current = current.Next {
        fmt.Print(current.Value, " ")
    }
    fmt.Println()
}

func main() {
    var list *Node
    list = list.Append(10)
    list = list.Append(20)
    list = list.Append(30)
    list.Walk() // 10 20 30
}
```

#### Example 3: Benchmarking Pointer vs. Value Receivers

```go
package counter

type Counter struct {
    val int
}

func (c Counter) Value() int {
    return c.val
}

func (c *Counter) Inc() {
    c.val++
}

// BenchmarkValueReceiver shows copy cost
func BenchmarkValueReceiver(b *testing.B) {
    c := Counter{}
    for i := 0; i < b.N; i++ {
        _ = c.Value()
    }
}

// BenchmarkPointerReceiver shows no copy
func BenchmarkPointerReceiver(b *testing.B) {
    c := &Counter{}
    for i := 0; i < b.N; i++ {
        _ = c.Value() // c is pointer, but Value has value receiver
    }
}
```

### 9. Summary & Exercises

#### Summary

- Go pointers are explicit memory addresses without arithmetic, balancing efficiency and safety.
- Escape analysis determines stack vs. heap placement – the compiler is aggressive but predictable.
- Use pointers for large structs (≥64 bytes), mutation, or to represent nil-able state.
- Avoid pointers for small values – they increase GC pressure and hurt CPU cache locality.
- Nil pointer dereference is the primary pointer-related bug; guard against it in public APIs.
- The garbage collector makes pointer use far safer than C/C++, but you still pay for heap allocations.

#### Exercises

**Exercise 1: Escape Analysis Prediction**

Given the following code, predict which variables escape to the heap. Then verify with `go build -gcflags="-m"`.

```go
type Item struct { Value int }

func build() []*Item {
    var out []*Item
    for i := 0; i < 10; i++ {
        item := Item{Value: i}
        out = append(out, &item)
    }
    return out
}
```

**Exercise 2: Optimize a Hot Path**

A service processes 1 million events per second. Each event is a `struct { ID int; Data [64]byte }`. The current code passes events by value to a logging function that never modifies them. Profile shows high GC pressure. Re-write to use pointers only where beneficial, and justify your changes.

**Exercise 3: Build a Thread-Safe Memory Pool**

Implement a `Pool[T any]` generic type that reuses pointers to `T` using `sync.Pool`. The pool should have methods `Get() *T` (returns a pointer to a zeroed `T` from pool or new one) and `Put(*T)`. Use escape analysis to ensure that using the pool actually reduces heap allocations. Write a benchmark proving the reduction.
