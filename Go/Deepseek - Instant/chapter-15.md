# Chapter 15: Generics (Type Parameters)

## Basic Usage

Go 1.18 introduced generics—arguably the most significant language change since Go 1.0. Type parameters allow you to write functions and data structures that operate on multiple types while maintaining compile-time type safety.

**Generic Functions:**

```go
// Basic generic function with a single type parameter
func Contains[T comparable](slice []T, item T) bool {
    for _, v := range slice {
        if v == item {
            return true
        }
    }
    return false
}

// Usage with type inference (most common)
numbers := []int{1, 2, 3, 4, 5}
fmt.Println(Contains(numbers, 3)) // true

strings := []string{"a", "b", "c"}
fmt.Println(Contains(strings, "d")) // false

// Explicit type parameter (rarely needed)
fmt.Println(Contains[int](numbers, 3))
```

**Generic Types (Structs):**

```go
// Generic stack implementation
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T // The zero value for type T
        return zero, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}

func (s *Stack[T]) Peek() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    return s.items[len(s.items)-1], true
}

// Usage
intStack := Stack[int]{}
intStack.Push(10)
intStack.Push(20)
val, ok := intStack.Pop() // val = 20, ok = true

stringStack := Stack[string]{}
stringStack.Push("hello")
```

**Multiple Type Parameters & Constraints:**

```go
// KeyValue pair with two type parameters
type Pair[K comparable, V any] struct {
    Key   K
    Value V
}

// Generic function to create a map from pairs
func ToMap[K comparable, V any](pairs []Pair[K, V]) map[K]V {
    result := make(map[K]V, len(pairs))
    for _, p := range pairs {
        result[p.Key] = p.Value
    }
    return result
}

// Custom constraint using interface
type Number interface {
    ~int | ~int64 | ~float64 // ~ means any type with underlying type
}

func Sum[T Number](numbers []T) T {
    var sum T
    for _, n := range numbers {
        sum += n
    }
    return sum
}

// Using type inference with custom types
type MyInt int
myNumbers := []MyInt{1, 2, 3, 4}
total := Sum(myNumbers) // Works due to ~int constraint
```

## Under the Hood

### Compilation Strategy: GCShape Stenciling

Go's generics implementation differs dramatically from C++ templates and Java erasure. The Go compiler uses a strategy called **GCShape stenciling** (also called "shape-based monomorphization"):

```go
// When you write:
func Identity[T any](t T) T { return t }

// The compiler doesn't generate code for every type. Instead, it:
// 1. Groups types by their "shape" (memory layout, method set)
// 2. Generates one implementation per distinct shape
// 3. Uses dictionary passing for method calls within the generic function
```

**Dictionary Passing:**

When a generic function calls methods on its type parameter, the compiler passes a **runtime dictionary** containing function pointers:

```go
type Stringer interface { String() string }

func Print[T Stringer](t T) {
    // The compiler generates a call through the dictionary
    fmt.Println(t.String()) // Dictionary contains pointer to T.String()
}
```

The dictionary contains:
- **Type metadata** (size, alignment, pointer maps for GC)
- **Method pointers** for each interface in the constraint
- **Comparison function** for `comparable` constraints

**Concrete Example of GCShape:**

```go
// All these types share the same shape (single pointer)
type User struct { Name string }     // shape: pointer to struct
type Product struct { ID int }       // shape: pointer to struct  
*User                                 // shape: pointer (same!)

// These have different shapes
int      // shape: integer
[3]int   // shape: array of 3 ints (different from slice)
[]int    // shape: slice header (ptr, len, cap)
```

### Performance Characteristics

1. **No Monomorphization Explosion:** Unlike C++, Go generates O(unique shapes) rather than O(unique types). If 100 different struct types are used with `Println`, C++ generates 100 copies; Go generates ~1-5 copies (grouped by shape).

2. **Dictionary Overhead:** Each generic function call passes an extra dictionary pointer (8 bytes). Method calls through the dictionary add indirection (similar to interface calls).

3. **Zero-cost for Non-Method Code:** If your generic function doesn't call methods on type parameters, the dictionary is only used for size/copy operations—minimal overhead.

```go
// Minimal overhead - just copies values
func Copy[T any](dst, src []T) int {
    return copy(dst, src) // copy is built-in, optimized
}

// More overhead - dictionary used for .String() method
func PrintAll[T fmt.Stringer](items []T) {
    for _, item := range items {
        fmt.Println(item.String()) // Indirect call through dict
    }
}
```

### Type Inference Implementation

Go's type inference works through **unification** (same algorithm used in Hindley-Milner type systems, but simplified):

```go
func Map[T, U any](slice []T, fn func(T) U) []U { ... }

// When called:
numbers := []int{1, 2, 3}
result := Map(numbers, func(i int) string { return strconv.Itoa(i) })
// Inference: T = int (from slice), U = string (from return type)
```

The algorithm:
1. Collects all type constraints from arguments
2. Unifies types (solving constraints like `[]T` must match `[]int`)
3. Checks constraint satisfaction at each unification step
4. Fails early with clear error messages

## Why This Design?

### Why Go Resisted Generics for So Long (Pre-1.18)

The Go team spent over a decade debating generics. The core tension: **simplicity vs. expressiveness**.

**Arguments against generics (pre-2018):**
- Adds complexity to the type system (Go prides itself on being learnable in a weekend)
- Increases compilation time (C++ template bloat was a cautionary tale)
- Encourages "abstraction for abstraction's sake" (Java-style generic factories)
- Most Go code simply didn't need them—`interface{}` and `go generate` covered many cases

**The tipping point:** By 2018, pain points became undeniable:
- Writing type-safe `Map/Filter/Reduce` required code generation
- Container libraries (e.g., `container/heap`) forced `interface{}` type assertions
- Database libraries couldn't provide type-safe row scanning
- The "avoid generics" dogma was hurting adoption in domains requiring strong typing

### Why Go Chose Stenciling Over Monomorphization (C++/Rust) or Erasure (Java)

**Rejection of Full Monomorphization (C++ approach):**
```cpp
// C++ generates code for EVERY instantiation
template<typename T>
T identity(T t) { return t; }

// Generates 4 separate functions in binary
identity<int>(42);      // identity_int
identity<double>(3.14); // identity_double  
identity<string>("hi"); // identity_string (copy semantics)
identity<User>(user);   // identity_User (could be huge)
```
- **Why Go rejected this:** Compilation time explosion, binary bloat, slow compile times
- Go's compilation speed is a non-negotiable feature for Google-scale monorepos

**Rejection of Type Erasure (Java approach):**
```java
// Java - type information lost at runtime
List<String> strings = new ArrayList<>();
List<Integer> ints = new ArrayList<>();
// strings.getClass() == ints.getClass() both return ArrayList.class
```
- **Why Go rejected this:** No runtime type information means `instanceof` limitations, array covariance issues, and inability to implement efficient generic containers without boxing
- Go's `any` is runtime-aware (interfaces carry type metadata)

**Why Go Chose GCShape + Dictionaries:**
- **Compilation speed:** O(shapes) rather than O(instantiations)
- **Binary size:** ~1-5 implementations per generic function
- **Runtime performance:** Dictionary overhead is comparable to interface calls (not zero-cost, but acceptable)
- **Type safety preserved:** All type information available at compile time AND runtime

### The `comparable` Constraint: A Philosophical Decision

Go is the only major language with a built-in `comparable` constraint. This reflects Go's pragmatic philosophy:

```go
// You cannot do this in most languages:
func Dedupe[T comparable](slice []T) []T {
    seen := make(map[T]bool)
    result := []T{}
    for _, v := range slice {
        if !seen[v] {
            seen[v] = true
            result = append(result, v)
        }
    }
    return result
}
```

**Why `comparable` exists:**
- Map keys require equality (`==` operator semantics)
- Go generics needed a way to express "T can be used as a map key"
- Rather than complex traits/type classes, Go provides a single built-in constraint

**What `comparable` includes:**
- All basic types (ints, strings, bools)
- Pointers (compares addresses)
- Arrays of comparable types
- Structs where all fields are comparable
- Interfaces (compares dynamic type and value)

**What's NOT comparable:**
- Slices (must use `reflect.DeepEqual` or `slices.Equal`)
- Maps
- Functions
- Structs containing slices/maps/functions

## Competing Approaches

### Go vs. C++ Templates

| Aspect | C++ Templates | Go Generics |
|--------|--------------|-------------|
| **Compilation model** | Full monomorphization (code per type) | GCShape stenciling (code per shape) |
| **Compilation time** | O(instantiations) - can be exponential | O(shapes) - linear in practice |
| **Binary size** | Potentially huge | Predictable growth |
| **Error messages** | Notorious for length and complexity | Clear, Go-style errors |
| **Specialization** | Yes (template specialization) | No (by design) |
| **Template metaprogramming** | Turing-complete | Not supported |
| **Concepts (C++20)** | Explicit constraints | Interface-based constraints |

**C++ approach wins when:** You need absolute zero-cost abstraction (embedded systems, HFT). Template metaprogramming enables compile-time computation.

**Go approach wins when:** Compilation speed matters (large monorepos, rapid iteration). You want error messages you can actually read.

### Go vs. Rust Generics

| Aspect | Rust | Go |
|--------|------|-----|
| **Monomorphization** | Always (full code generation) | Sometimes (GCShape with dictionaries) |
| **Zero-cost abstraction** | Yes (no runtime overhead) | Partial (dictionary for methods) |
| **Trait bounds** | Comprehensive | Interface-based constraints |
| **Associated types** | Yes | No (use multiple type params) |
| **Higher-kinded types** | No (but planned) | No |
| **Specialization** | Unstable | No |
| **Compile times** | Slow (one of Rust's pain points) | Fast (Go's priority) |

**Example comparison:**

```rust
// Rust - monomorphized, zero-cost
fn identity<T>(t: T) -> T { t }

// Can express complex bounds
fn process<T: Clone + std::fmt::Debug + Iterator>(item: T) { ... }
```

```go
// Go - dictionary passing for methods
func Identity[T any](t T) T { return t }

// Multiple constraints via interface composition
type Processable[T any] interface {
    Clone() T
    fmt.Stringer
    ~[]T // Underlying type constraint
}

func Process[T Processable[U], U any](item T) { ... }
```

**Rust wins:** Maximum performance, fine-grained control over memory layout, no hidden overhead.

**Go wins:** Compilation speed (5-10x faster than Rust for generic-heavy code), simpler mental model.

### Go vs. Java Generics

| Aspect | Java | Go |
|--------|------|-----|
| **Implementation** | Type erasure | Reified (type info preserved) |
| **Runtime type info** | Lost (except for bounds) | Available via reflection |
| **Primitive support** | Boxing required | Direct (no boxing) |
| **Array covariance** | Broken (runtime errors) | Type-safe |
| **Wildcard types** | Yes (`? extends T`) | No (use interfaces) |
| **Type inference** | Limited (Java 7+) | Robust (unification) |

**Java's critical weakness (runtime type loss):**
```java
// This fails at runtime due to erasure
List<String> strings = new ArrayList<>();
List<Integer> ints = new ArrayList<>();
if (strings.getClass() == ints.getClass()) { // true!
    // Type information completely gone
}
```

**Go's advantage:**
```go
var strings []string
var ints []int
// reflect.TypeOf(strings) != reflect.TypeOf(ints) - different types
// Generic types carry their type parameters at runtime
type List[T any] []T
slist := List[string]{}
// reflect.TypeOf(slist).String() returns "main.List[string]"
```

## Common Mistakes

### Mistake 1: Over-abstracting Too Early

**Wrong:**
```go
// Don't start with generics
type Repository[T any] interface {
    Get(id string) (T, error)
    Save(t T) error
}

// This forces EVERY repository into the same shape
// But UserRepo needs different methods (FindByEmail)
// ProductRepo needs different methods (FindByCategory)
```

**Right:**
```go
// Start concrete, abstract when you have 3+ implementations
type UserRepository interface {
    Get(id string) (*User, error)
    FindByEmail(email string) (*User, error)
    Save(user *User) error
}

// Only use generics when behavior is truly type-agnostic
type Cache[T any] struct {
    store map[string]T
    mu    sync.RWMutex
}

func (c *Cache[T]) Get(key string) (T, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    val, ok := c.store[key]
    return val, ok
}
```

### Mistake 2: Forgetting That Methods Can't Have Type Parameters

**Wrong:**
```go
type Wrapper[T any] struct {
    Value T
}

// This DOES NOT compile - methods can't introduce new type params
func (w *Wrapper[T]) Map[U any](fn func(T) U) Wrapper[U] {
    return Wrapper[U]{Value: fn(w.Value)}
}
```

**Right:**
```go
// Make it a function instead
func Map[T, U any](w Wrapper[T], fn func(T) U) Wrapper[U] {
    return Wrapper[U]{Value: fn(w.Value)}
}

// Or store the function in the type
type Mapper[T, U any] struct {
    Wrapper[T]
    Fn func(T) U
}

func (m *Mapper[T, U]) Map() Wrapper[U] {
    return Wrapper[U]{Value: m.Fn(m.Wrapper.Value)}
}
```

### Mistake 3: Misunderstanding the `~` Tilde

**Wrong:**
```go
type Integer interface {
    int | int64 // ONLY works with exactly int or int64
}

type MyInt int
total := Sum([]MyInt{1,2,3}) // COMPILE ERROR!
```

**Right:**
```go
type Integer interface {
    ~int | ~int64 // ~ means "any type with underlying type int or int64"
}

type MyInt int
total := Sum([]MyInt{1,2,3}) // Works fine
```

### Mistake 4: Using `any` Constraint When You Need Operations

**Wrong:**
```go
// This compiles but fails when used
func Max[T any](a, b T) T {
    if a > b { // COMPILE ERROR: any has no > operator
        return a
    }
    return b
}
```

**Right:**
```go
// Define what operations you need
type Ordered interface {
    ~int | ~int64 | ~float64 | ~string
}

func Max[T Ordered](a, b T) T {
    if a > b {
        return a
    }
    return b
}
```

### Mistake 5: Generic Channel Types Without Careful Ownership

**Wrong:**
```go
type Pipeline[T any] struct {
    in  chan T
    out chan T
}

func (p *Pipeline[T]) Process() {
    // Dangerous - who closes the channels?
    for v := range p.in {
        p.out <- transform(v)
    }
    // If we close p.out here, but caller might still write?
}
```

**Right:**
```go
// Document ownership clearly
type Pipeline[T any] struct {
    in  <-chan T   // Receive-only, caller owns
    out chan<- T   // Send-only, we own
}

func (p *Pipeline[T]) Process(ctx context.Context) {
    defer close(p.out) // We own out, so we close it
    
    for {
        select {
        case <-ctx.Done():
            return
        case v, ok := <-p.in:
            if !ok {
                return // Input closed, we're done
            }
            p.out <- transform(v)
        }
    }
}
```

## Performance Considerations

### Memory Allocation Patterns

```go
// BAD: Forces heap allocation for zero values
func FindFirst[T any](slice []T, predicate func(T) bool) (T, bool) {
    for _, v := range slice {
        if predicate(v) {
            return v, true
        }
    }
    var zero T // This MAY allocate if T contains pointers
    return zero, false
}

// GOOD: Use pointer return to avoid value copy
func FindFirstPtr[T any](slice []T, predicate func(T) bool) (*T, bool) {
    for i := range slice {
        if predicate(slice[i]) {
            return &slice[i], true
        }
    }
    return nil, false
}
```

### Benchmark Comparison: Generic vs. Interface vs. Concrete

```go
type Identifiable interface { ID() int }

type User struct { id int }
func (u User) ID() int { return u.id }

// Concrete version (fastest, but not reusable)
func FindUserByID(users []User, target int) (User, bool) {
    for _, u := range users {
        if u.id == target {
            return u, true
        }
    }
    return User{}, false
}

// Interface version (slowest due to dynamic dispatch)
func FindByIDInterface(items []Identifiable, target int) (Identifiable, bool) {
    for _, item := range items {
        if item.ID() == target {
            return item, true
        }
    }
    return nil, false
}

// Generic version (middle ground)
func FindByIDGeneric[T Identifiable](items []T, target int) (T, bool) {
    for _, item := range items {
        if item.ID() == target {
            return item, true
        }
    }
    var zero T
    return zero, false
}

// Benchmark results (typical):
// Concrete:    1.2 ns/op, 0 allocs
// Generic:     2.8 ns/op, 0 allocs  (2.3x slower)
// Interface:   5.1 ns/op, 0 allocs  (4.2x slower)
```

### GC Pressure with Generic Containers

```go
// BAD: Generic container storing pointers causes more GC work
type PointerCache[T any] struct {
    items []*T  // Each pointer needs scanning
}

// GOOD: Store values directly when possible
type ValueCache[T any] struct {
    items []T   // Scanning depends on T's contents
}

// For large T, use pointer but batch operations
type BatchProcessor[T any] struct {
    batch []T
    threshold int
}

func (b *BatchProcessor[T]) Add(item T) {
    b.batch = append(b.batch, item)
    if len(b.batch) >= b.threshold {
        b.process()
    }
}
```

### Escape Analysis with Generics

```go
// This function may cause heap allocation
func Identity[T any](t T) T {
    // If T is large (>10 words), t may escape to heap
    return t
}

// How to check:
// go build -gcflags="-m" main.go
// Output: ./main.go:5:7: leaking param: t to result ~r0 level=0

// For large types, consider pointer receivers
type LargeStruct struct {
    data [1024]int
}

func (l *LargeStruct) Process[T any](fn func(*LargeStruct) T) T {
    // l stays on heap, but that's fine - it's already there
    return fn(l)
}
```

## Best Practices

### 1. Accept Interfaces, Return Structs (Even with Generics)

```go
// BAD: Returning generic interface
func NewProcessor[T any]() *Processor[T] {
    return &Processor[T]{}
}

// GOOD: Keep generics in the struct, but accept interfaces
type Processor[T any] struct {
    handlers []func(T) error
}

func (p *Processor[T]) AddHandler(handler func(T) error) {
    p.handlers = append(p.handlers, handler)
}

// Accept interfaces for callbacks
func (p *Processor[T]) ProcessWithFilter(ctx context.Context, 
                                         items []T, 
                                         filter interface{ Filter(T) bool }) {
    // Filter is an interface, T is concrete in struct
}
```

### 2. Use Type Inference - Don't Explicitly Specify Types

```go
// UGLY: Unnecessary explicit types
result := Map[int, string]([]int{1,2,3}, func(i int) string { 
    return strconv.Itoa(i) 
})

// CLEAN: Let inference work
result := Map([]int{1,2,3}, func(i int) string { 
    return strconv.Itoa(i) 
})
```

### 3. Keep Constraints Small and Focused

```go
// BAD: God interface constraint
type Serializable interface {
    fmt.Stringer
    json.Marshaler
    encoding.BinaryMarshaler
    ~[]byte | ~string
    comparable
}

// GOOD: Single responsibility constraints
type Stringable interface { String() string }
type Encodable interface { Encode() ([]byte, error) }

// Compose when needed
func Save[T Stringable & Encodable](t T) error {
    // Only requires what it actually uses
}
```

### 4. Name Type Parameters Meaningfully

```go
// BAD: Cryptic names
func Map[T, U, V any](fn func(T) (U, V)) []V { ... }

// GOOD: Descriptive names
func Map[Input, Output1, Output2 any](
    transform func(Input) (Output1, Output2),
) []Output2 { ... }

// Conventional single-letter (for simple cases):
// T - any type
// K - map key (must be comparable)  
// V - map value
// E - element type
func Keys[K comparable, V any](m map[K]V) []K { ... }
```

### 5. Prefer Functions Over Methods for Type Conversion

```go
// BAD: Method with type parameters (impossible)
type Slice[T any] []T
func (s Slice[T]) Map[U any](fn func(T) U) Slice[U] // ERROR

// GOOD: Top-level function
func Map[T, U any](s []T, fn func(T) U) []U {
    result := make([]U, len(s))
    for i, v := range s {
        result[i] = fn(v)
    }
    return result
}
```

## Examples

### Example 1: Generic LRU Cache

```go
package cache

import (
    "container/list"
    "sync"
)

// LRU implements a thread-safe LRU cache with generics
type LRU[K comparable, V any] struct {
    capacity int
    cache    map[K]*list.Element
    list     *list.List
    mu       sync.RWMutex
}

type entry[K comparable, V any] struct {
    key   K
    value V
}

func NewLRU[K comparable, V any](capacity int) *LRU[K, V] {
    if capacity <= 0 {
        capacity = 100 // sensible default
    }
    return &LRU[K, V]{
        capacity: capacity,
        cache:    make(map[K]*list.Element, capacity),
        list:     list.New(),
    }
}

func (l *LRU[K, V]) Get(key K) (V, bool) {
    l.mu.Lock()
    defer l.mu.Unlock()
    
    if elem, ok := l.cache[key]; ok {
        l.list.MoveToFront(elem)
        return elem.Value.(*entry[K, V]).value, true
    }
    
    var zero V
    return zero, false
}

func (l *LRU[K, V]) Put(key K, value V) {
    l.mu.Lock()
    defer l.mu.Unlock()
    
    if elem, ok := l.cache[key]; ok {
        l.list.MoveToFront(elem)
        elem.Value.(*entry[K, V]).value = value
        return
    }
    
    if l.list.Len() >= l.capacity {
        l.evict()
    }
    
    elem := l.list.PushFront(&entry[K, V]{key: key, value: value})
    l.cache[key] = elem
}

func (l *LRU[K, V]) evict() {
    elem := l.list.Back()
    if elem != nil {
        l.list.Remove(elem)
        delete(l.cache, elem.Value.(*entry[K, V]).key)
    }
}

func (l *LRU[K, V]) Len() int {
    l.mu.RLock()
    defer l.mu.RUnlock()
    return l.list.Len()
}

// Usage
func main() {
    cache := NewLRU[string, int](3)
    cache.Put("a", 1)
    cache.Put("b", 2)
    cache.Put("c", 3)
    
    val, ok := cache.Get("b") // val=2, ok=true
    cache.Put("d", 4)          // evicts "a"
}
```

### Example 2: Generic Result Type (Rust-style)

```go
package result

import (
    "errors"
    "fmt"
)

type Result[T any, E error] struct {
    value T
    err   E
    isErr bool
}

func Ok[T any, E error](value T) Result[T, E] {
    return Result[T, E]{value: value, isErr: false}
}

func Err[T any, E error](err E) Result[T, E] {
    var zero T
    return Result[T, E]{value: zero, err: err, isErr: true}
}

func (r Result[T, E]) IsOk() bool {
    return !r.isErr
}

func (r Result[T, E]) IsErr() bool {
    return r.isErr
}

func (r Result[T, E]) Unwrap() T {
    if r.isErr {
        panic("called Unwrap on an Err value")
    }
    return r.value
}

func (r Result[T, E]) UnwrapOr(defaultVal T) T {
    if r.isErr {
        return defaultVal
    }
    return r.value
}

func (r Result[T, E]) UnwrapOrElse(fn func(E) T) T {
    if r.isErr {
        return fn(r.err)
    }
    return r.value
}

func (r Result[T, E]) Map[U any](fn func(T) U) Result[U, E] {
    if r.isErr {
        var zero U
        return Result[U, E]{value: zero, err: r.err, isErr: true}
    }
    return Ok[U, E](fn(r.value))
}

func (r Result[T, E]) AndThen[U any](fn func(T) Result[U, E]) Result[U, E] {
    if r.isErr {
        var zero U
        return Result[U, E]{value: zero, err: r.err, isErr: true}
    }
    return fn(r.value)
}

// Example usage
func parseInt(s string) Result[int, error] {
    var err error
    // Parse logic
    val := 0
    for _, c := range s {
        if c < '0' || c > '9' {
            err = errors.New("invalid number")
            break
        }
        val = val*10 + int(c-'0')
    }
    if err != nil {
        return Err[int, error](err)
    }
    return Ok[int, error](val)
}

func main() {
    result := parseInt("123")
    doubled := result.Map(func(x int) int { return x * 2 })
    
    val := doubled.UnwrapOr(0) // 246
    fmt.Println(val)
}
```

## Summary

**Key Takeaways:**

1. **Go's generics prioritize compilation speed** over runtime performance, using GCShape stenciling rather than full monomorphization (C++) or type erasure (Java).

2. **The `comparable` constraint** is Go's unique solution to the "map key problem" - simpler than Rust's traits or Haskell's type classes.

3. **Methods cannot introduce new type parameters** - this forces you to use top-level functions for transformations, keeping the language simpler.

4. **Performance trade-off:** Generic functions have dictionary overhead for method calls (~2-3x slower than concrete, but 2x faster than interfaces).

5. **The tilde operator (`~int`)** is critical for working with custom types that have underlying built-in types.

6. **When to use generics:** 
   - Container data structures (lists, maps, caches)
   - Algorithms operating on multiple types (sort, filter, map)
   - Type-safe wrappers around untyped APIs (JSON, database rows)

7. **When NOT to use generics:**
   - Single concrete implementation (just use the concrete type)
   - Different method signatures (use interfaces)
   - When type assertions would be simpler (don't over-engineer)

**Philosophy Connection:** Generics represent Go's careful balance - adding expressive power without sacrificing the simplicity that makes Go readable and maintainable. Unlike C++ templates which enable a "language within a language," Go's generics are intentionally limited: no template metaprogramming, no partial specialization, no variadic type parameters. This is the "less is more" philosophy applied to one of the most complex areas of language design.

## Exercises

### Exercise 1: Generic Pipeline with Error Handling

Build a generic `Pipeline` type that processes data through stages, where each stage can fail. Implement:

```go
type Pipeline[T any] struct {
    stages []func(T) (T, error)
}

func NewPipeline[T any]() *Pipeline[T]

func (p *Pipeline[T]) AddStage(fn func(T) (T, error)) *Pipeline[T]

func (p *Pipeline[T]) Process(initial T) (T, error)
```

**Challenge:** Add support for parallel processing of stages where order doesn't matter, using goroutines and proper error aggregation.

### Exercise 2: Generic Comparator for Complex Sorting

Implement a type-safe comparator system that can sort slices by multiple fields:

```go
type Comparator[T any] func(a, b T) int

func By[T, F any](extract func(T) F) Comparator[T] // where F must be Ordered

func Then[T any](first, second Comparator[T]) Comparator[T]

func Sort[T any](slice []T, cmp Comparator[T])
```

**Requirements:**
- Support chaining comparators (e.g., `Sort(users, By(User.ID).Then(By(User.Name)))`)
- Handle nil values gracefully
- Provide ascending/descending modifiers

**Test:** Sort a slice of structs with 4 fields, falling back to next field when previous fields are equal.

### Exercise 3: Generic Event Bus with Type Safety

Create a type-safe event bus where each event type has its own channel:

```go
type EventBus struct {
    // Must store subscribers of different types
}

func (e *EventBus) Subscribe[T any](fn func(T)) *Subscription

func (e *EventBus) Publish[T any](event T)

type Subscription struct {
    cancel func()
}

func (s *Subscription) Unsubscribe()
```

**Constraints:**
- Subscribers should only receive events of their declared type
- Publishing the wrong type should be a compile-time error
- Must handle concurrent Subscribe/Publish safely
- Unsubscribing should not affect other subscribers

**Advanced:** Add support for wildcard subscriptions (`any`) and topic-based filtering using struct tags or reflection.
