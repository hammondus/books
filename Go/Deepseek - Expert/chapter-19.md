# Chapter 19: Pointers & Memory Allocation

Go’s pointer model occupies a deliberate middle ground: it gives you the low-level power of indirection without the sharp edges of pointer arithmetic or manual memory management. For the seasoned engineer, that means writing code that’s both performant and safe, while the compiler shoulders most of the memory-layout decisions. In this chapter we’ll explore how Go pointers really work, how the runtime decides between stack and heap, and how to make practical optimisation decisions that don’t sacrifice clarity.

---

## 1. Basic Usage

At the syntax level, Go pointers look familiar: `*T` is a pointer to a value of type `T`, the `&` operator takes the address of a variable, and `*` dereferences a pointer. There is no pointer arithmetic, no `->` operator, and no explicit deallocation.

```go
package main

import "fmt"

type Config struct {
    Timeout int
    Retries int
}

// modify mutates the Config pointed to by c.
func modify(c *Config) {
    if c == nil {
        return // guard against nil
    }
    c.Timeout = 30
}

func main() {
    var cfg Config           // zero value
    modify(&cfg)             // pass address
    fmt.Println(cfg.Timeout) // 30

    // Pointer from composite literal
    cfgPtr := &Config{Timeout: 10, Retries: 3}
    modify(cfgPtr)
    fmt.Println(cfgPtr.Timeout) // 30

    // The new function allocates zero value, returns pointer.
    p := new(int)
    *p = 42
    fmt.Println(*p) // 42
}
```

A pointer’s zero value is `nil`. Dereferencing a nil pointer causes a run‑time panic, so defensive checks (or clear invariants that a pointer is never nil) are essential. Struct fields, array elements, and local variables can all be addressed, but you cannot take the address of a map element (maps may rehash) or a constant.

Pointers are typed: a `*int` is distinct from a `*int32`; there is no `void*`. The `unsafe.Pointer` type exists for low‑level interop but we’ll cover that in Chapter 32. For everyday Go, typed pointers preserve memory safety.

A crucial nuance: **everything in Go is passed by value**. When you pass a pointer to a function, you pass a *copy of the address*. The callee receives a new pointer variable that points to the same memory. This means reassigning the pointer itself inside the function does not affect the caller’s pointer, although mutating the target does.

```go
func reassign(p *int) {
    x := 99
    p = &x // only changes local copy of p
}
```

To change which object a caller’s pointer refers to, you would need a pointer to a pointer, but that pattern is rare. Usually you mutate through the pointer.

---

## 2. Under the Hood

Go’s runtime represents a pointer as a machine word containing a memory address. The compiler tracks the type of the pointed‑to memory, enabling the garbage collector to traverse the object graph precisely. Unlike C, a pointer never disguises itself as an integer; the GC can always tell which words are pointers (with help from the compiler’s stack maps and type descriptors).

When you write `var x int = 10; p := &x`, the compiler must decide where `x` lives: on the goroutine’s stack or on the heap. That decision is made by **escape analysis**, a static analysis pass that determines whether a variable’s lifetime could outlive the function that creates it. If the analysis proves the variable does not escape, it is allocated on the stack; otherwise it “escapes to the heap.”

The stack is a per‑goroutine region that grows and shrinks as functions are called. Allocating on the stack is simply a matter of moving the stack pointer; it costs a few CPU instructions and no GC overhead. The heap is a shared, garbage‑collected area. Allocating there involves finding free space, and every heap allocation adds work for the GC later.

Escape analysis is conservative: if the compiler cannot prove a variable does not escape, it places it on the heap. Common escape causes include:

- Returning a pointer to a local variable.
- Storing a value into an interface (the value is boxed and typically escapes).
- Assigning a pointer to a global variable or a struct field reachable from outside.
- Sending a pointer through a channel or storing it in a slice that itself escapes.

You can inspect the compiler’s decisions with:

```
go build -gcflags="-m" .
```

A line like `moved to heap: x` tells you that `x` escaped. Understanding these reports is the first step toward writing allocation‑conscious Go.

Finally, pointer indirection carries a microscopic runtime cost: the CPU must load the address from memory before accessing the target, potentially missing the data cache. However, for all but the hottest loops, that cost is negligible compared to the cost of copying large structures.

---

## 3. Why This Design?

The Go team chose to include explicit pointers while forbidding pointer arithmetic and manual memory management. Why not omit pointers altogether, as Java and Python did? Why not provide full control, as C and C++ do?

The answer lies in the “Less is More” philosophy. Pointers solve two practical problems:

1. **Mutability without copying.** Without pointers, you would have to copy a struct to modify it and then return the new value, which can be unnatural for data that is inherently stateful (a cache, a connection pool, a buffer). Pointers let a function modify data in place, keeping the API natural.

2. **Performance.** Copying large structs in every function call would quickly dominate CPU time and memory traffic. A pointer is a single word, so passing it is cheap.

Yet the Go team deliberately removed pointer arithmetic. That decision eliminates entire classes of bugs—buffer overflows, use‑after‑free, pointer type confusion—that have plagued C code for decades. The absence of arithmetic also makes precise garbage collection possible, because the GC never has to guess whether an arbitrary integer is actually a pointer.

The explicit syntax (`*T`, `&`) makes the cost of indirection visible. When you see `*Config`, you know the function may modify the underlying value and that a level of indirection exists. In Java, every non‑primitive variable is a reference, but the syntax hides that; you cannot see whether an assignment shares or copies data. Go makes that choice explicit, which aligns with its goal of code being clear and straightforward.

The lack of manual memory management is another deliberate trade‑off. The team decided that the productivity and safety gained from garbage collection outweigh the loss of deterministic deallocation for the vast majority of programs they were building at Google. For the rare cases that need arena‑style allocation, Go 1.20 introduced experimental arenas (though they may be removed in the future), and you can always drop into `cgo` or `unsafe`. By default, however, you let the GC do its job.

In short, Go’s pointers are a pragmatic compromise: enough power to write systems software, but not enough rope to hang yourself—or the runtime.

---

## 4. Competing Approaches

**C / C++**
C gives you raw pointers, pointer arithmetic, and `malloc`/`free`. C++ adds references, smart pointers (`unique_ptr`, `shared_ptr`), and move semantics. Both languages prioritise absolute control over memory layout and lifetime. The cost is a much higher risk of memory corruption, leaks, and undefined behaviour. Go trades that control for safety and rapid development while still allowing pointer indirection.

**Rust**
Rust also has pointers (references) but wraps them in an ownership system with borrow checking. References in Rust are statically guaranteed to be valid and free of data races. Go does not attempt to prove safety at compile time; instead it relies on a runtime (GC and race detector). Rust’s model yields zero‑cost memory safety without a GC, but at the expense of a steep learning curve. Go’s model is simpler to learn and reason about, at the cost of GC pauses and some runtime bookkeeping.

**Java / C#**
These languages have only references, no explicit pointers (outside of `unsafe` blocks). All non‑primitive types are allocated on the heap and garbage collected. The programmer cannot choose stack allocation for an object, and every object variable is a reference, which hides the sharing. Go’s ability to allocate value types on the stack and use explicit pointers gives finer control over memory layout and reduces GC pressure.

**Python**
Python’s “everything is an object” means every variable is a reference. There is no concept of a value type or a pointer; you cannot have a plain struct on the stack. Go’s value types and pointers make it possible to write high‑throughput, low‑allocation code that Python cannot easily match.

Go’s approach thus sits between the managed‑only worlds of Java/Python and the manual‑control worlds of C/C++/Rust. It provides a **tunable** mechanism: you start with values on the stack, and only when necessary you introduce pointers that may trigger heap allocation. The escape analysis tooling lets you observe and optimise this boundary.

---

## 5. Common Mistakes

**1. Nil pointer dereference**
The most frequent panic in Go. A pointer variable, struct field, or return value that is not initialised is `nil`. Always guard with `if p == nil` before dereferencing, or design invariants so nil cannot occur.

```go
func update(cfg *Config) {
    // BAD: panics if cfg is nil
    cfg.Timeout = 10
}
```

**2. Passing pointers to slices, maps, or channels**
Slices, maps, and channels already contain internal pointers (a slice has a pointer to the backing array). Passing a `*[]int` is virtually never necessary and adds an unnecessary layer of indirection. The callee can already mutate the underlying data via the slice header; if you need to change the slice header itself (for example, to replace the whole slice), consider returning a new slice instead.

**3. Taking the address of a loop variable (pre‑1.22)**
Before Go 1.22, the loop variable was reused across iterations, so capturing its address led to all pointers pointing to the same memory (the final iteration’s value). Go 1.22 fixed this by making each iteration have its own variable, but old code patterns and older compiler versions still exhibit this bug. Always be explicit:

```go
for i := range items {
    i := i // create new variable (still idiomatic for clarity)
    go process(&i)
}
```

**4. Pointer to interface**
An interface value already holds a pointer to the underlying concrete data. If you declare `var w io.Writer = &bytes.Buffer{}`, `w` itself is an interface value containing a pointer to the buffer. Declaring `*io.Writer` is almost always a mistake; it adds a level of indirection that is rarely needed and may cause confusion. Accept interfaces by value; the concrete type can hold a pointer internally.

**5. Inadvertent escapes**
A local variable that “should” be on the stack may escape because of a seemingly innocent interface conversion or closure. For example:

```go
func sum(vals []int) int {
    total := 0
    for _, v := range vals {
        total += v
    }
    return total
}
// total stays on the stack.

func sumAny(vals []interface{}) interface{} {
    total := 0
    for _, v := range vals {
        total += v.(int) // type assertion may cause boxing
    }
    return total // total escapes because it's returned as interface
}
```

Often the fix is to design away the interface, or to accept the heap allocation when it simplifies the architecture.

---

## 6. Performance Considerations

**Cost of indirection**
Dereferencing a pointer loads a cache line; if the target is not already in cache, you pay a main‑memory access. For small, frequently accessed data, keeping it inline (by value) is often faster. Always measure: the difference is negligible in most code, but hot paths benefit from cache‑friendly layouts.

**Copying vs. pointer passing**
Passing a `struct` by value copies every field. A struct with a few machine words (e.g., up to ~64 bytes) copies about as fast as passing a pointer, because the copy uses contiguous memory and the pointer itself still requires memory access. For larger structs, copying cost grows linearly. The pragmatic threshold varies, but common idioms dictate: use a pointer when the struct is “large” (subjective, but typically > 100–200 bytes) or when you need mutability. The `strings.Builder` is a notable exception: it must be passed by pointer because it is never safe to copy (contains internal state).

**Allocation counts**
Every heap allocation adds work for the garbage collector. A function that returns a pointer forces the pointed‑to value onto the heap (unless it can be proved that the pointer does not escape, which is rare). By contrast, returning a value keeps it on the stack, avoiding an allocation. When prototyping, heap allocations are fine. When optimising, use benchmarks and `-gcflags="-m"` to identify hot allocations.

**Interface boxing**
Assigning a concrete value to an interface variable creates an interface value that holds a copy of the data. If the data is a large struct, that copy may be expensive and may cause an escape. Storing a pointer inside the interface (i.e., the concrete type is already a pointer, like `*bytes.Buffer`) avoids copying the whole struct; the interface then holds just the pointer word. This is why many types that satisfy interfaces are used via pointers.

**GC pressure**
The GC’s mark phase must traverse every live pointer from roots (stacks, globals) into the heap. More pointers mean more work. Keeping data on the stack or embedding values directly inside other structs reduces the number of scanned objects. Using `sync.Pool` to reuse heap‑allocated objects also curbs allocations.

**Escape analysis tips**
- Avoid returning pointers to local variables if you can return the value instead.
- When possible, use value types rather than pointers for temporary data that does not need identity.
- Inline functions may give the compiler more visibility to prove an allocation does not escape.

Profile before changing your design: `go test -bench . -benchmem` shows allocations per operation.

---

## 7. Best Practices

1. **Use a pointer when you need to mutate the caller’s data, or when the struct is large.** For small, immutable types (`int`, `bool`, `string`, small structs), pass by value.

2. **Be consistent with method receivers.** If any method on a type requires a pointer receiver (for mutability or to avoid copying), it is idiomatic to make all methods pointer receivers, even those that could technically use a value receiver. This keeps the method set consistent and avoids surprises with interface satisfaction.

3. **Check for nil at the boundary.** Every public function or method that accepts a pointer should define whether `nil` is valid. If it is not valid, document it and guard with a panic or early return. The standard library often panics with a clear message (e.g., `net/http`), which is acceptable for programmer errors.

4. **Avoid `new(T)`; prefer `&T{}`.** The latter is more flexible because it can include field initialisation. Both allocate a zeroed value and return a pointer; the choice is a matter of style, but the composite literal form is more common.

5. **Never use pointer to interface.** Accept interfaces by value. If you need to modify an interface’s target, put a pointer *inside* the interface (i.e., the concrete type is a pointer).

6. **Use `go vet` to detect nil dereference paths and other suspicious pointer usage.** The race detector (`-race`) will catch unsynchronised access to shared memory through pointers.

7. **Let escape analysis guide you, but don’t obsess.** Write clear, correct code first. If profiling reveals a bottleneck, consult the escape report and consider adjustments. “Premature optimisation is the root of all evil” holds in Go as much as anywhere.

8. **Avoid pointer fields for data that logically belongs inline.** If a struct always needs a `Config` and `Config` is small, embed it by value; the memory locality improves and the GC has fewer objects to scan.

9. **When sharing data between goroutines, prefer channels over passing raw pointers**, following the mantra “share memory by communicating.” If you do pass pointers across goroutines, protect access with a mutex or use `sync/atomic` for simple types. The race detector is your friend.

---

## 8. Examples

**Example 1: Escaping analysis in practice**
The following program uses a `bytes.Buffer` locally and returns a pointer to one of its fields. The compiler’s escape report shows what moves to the heap.

```go
package main

import "bytes"

type Report struct {
    Title string
    Body  string
}

func generate() *Report {
    var buf bytes.Buffer
    buf.WriteString("Annual Report")
    // buf is used locally, but does it escape?
    r := &Report{
        Title: buf.String(),
        Body:  "some body",
    }
    return r // r escapes to heap
}

func main() {
    _ = generate()
}
```

Build with `-m`:
```
$ go build -gcflags="-m" .
./main.go:11:6: can inline generate
./main.go:12:15: inlining call to bytes.(*Buffer).WriteString
./main.go:15:9: &Report{...} escapes to heap
```
The `Report` composite literal escapes because it is returned as a pointer. The `buf` itself does not escape; it lives on the stack.

**Example 2: Benchmarking value vs. pointer for a moderate struct**

```go
package bench

import "testing"

type Data struct {
    ID    int
    Name  string
    Flags [16]bool
    Meta  [128]byte
}

//go:noinline
func processValue(d Data) int {
    return d.ID + int(d.Meta[0])
}

//go:noinline
func processPointer(d *Data) int {
    return d.ID + int(d.Meta[0])
}

var sink int

func BenchmarkValue(b *testing.B) {
    d := Data{ID: 42}
    for i := 0; i < b.N; i++ {
        sink = processValue(d)
    }
}

func BenchmarkPointer(b *testing.B) {
    d := &Data{ID: 42}
    for i := 0; i < b.N; i++ {
        sink = processPointer(d)
    }
}
```

Results will show that passing a value of this size incurs a larger copy cost; the pointer version avoids the copy but adds indirection. The exact crossover point depends on CPU architecture and cache state, which is why we benchmark.

**Example 3: Nil‑safe method receiver**

```go
type Cache struct {
    mu    sync.Mutex
    items map[string]string
}

// Get safely handles a nil receiver.
func (c *Cache) Get(key string) (string, bool) {
    if c == nil {
        return "", false
    }
    c.mu.Lock()
    defer c.mu.Unlock()
    v, ok := c.items[key]
    return v, ok
}
```

This pattern is common in the standard library (e.g., `http.ServeMux`) and allows a zero‑value `*Cache` to behave like an empty cache without panicking.

---

## 9. Summary & Exercises

**Key takeaways:**
- Pointers in Go provide explicit indirection, no arithmetic, and are garbage collected.
- Everything is passed by value; a pointer parameter receives a copy of the address.
- Escape analysis decides stack vs. heap; `-gcflags="-m"` reveals its choices.
- Use pointers for mutability or large structs; use values for small, immutable data and better cache locality.
- Common pitfalls include nil dereference, pointer‑to‑interface, and inadvertent heap escapes.
- Benchmark before trading clarity for allocations; the “Go way” prefers simplicity first.

**Exercises:**

1. **Thread‑safe cache with pointer semantics**
   Build a generic cache `type Cache[K comparable, V any]` that stores key‑value pairs. Use a `sync.RWMutex` and a `map[K]V` inside. Provide methods `Get`, `Set`, and `Delete` on `*Cache`. Ensure `nil` receiver is handled safely. Write benchmarks that compare pointer‑receiver vs. value‑receiver (on a struct containing a mutex; value receiver would copy the mutex, leading to a `go vet` warning—explain why value receiver is dangerous). Analyse escape behaviour of the values stored.

2. **Minimising allocations**
   Given a function that processes a large log line and extracts fields, returning a slice of strings and a struct of metadata, identify which allocations escape and refactor to keep them on the stack where possible. Use `-benchmem` to track progress. Consider using `strings.Builder` and passing it by pointer.

3. **Pointer‑sharing and the race detector**
   Write a program that launches several goroutines, each receiving a pointer to a shared counter struct (containing a `sync.Mutex` and an `int`). Increment the counter concurrently, once using proper mutex locking, and once intentionally without (to trigger the race detector). Run with `-race` and interpret the output. Discuss how “share memory by communicating” could replace the raw pointer with a channel‑based design.

---
