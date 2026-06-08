## Chapter 15: Generics (Type Parameters)

Generics arrived in Go 1.18 after a decade of deliberate restraint. This chapter is not an apology for their absence, nor a celebration of their arrival. It is an engineer’s guide to when, how, and—critically—*when not* to use type parameters in a language that already solved many abstraction problems through interfaces.

---

### 1. Basic Usage

A **generic function** declares one or more type parameters inside square brackets before the regular parameter list:

```go
// Max returns the larger of two ordered values.
func Max[T cmp.Ordered](a, b T) T {
    if a > b {
        return a
    }
    return b
}
```

`cmp.Ordered` is an interface constraint that permits the `>` operator. Invocation can use explicit type arguments, but Go usually infers them:

```go
x := Max(3, 5)          // T inferred as int
y := Max("apple", "z")  // T inferred as string
```

A **generic type** parameterises a struct, slice, or map:

```go
type Set[T comparable] map[T]struct{}

func NewSet[T comparable](vals ...T) Set[T] {
    s := make(Set[T])
    for _, v := range vals {
        s[v] = struct{}{}
    }
    return s
}
```

**Constraints** are interfaces that describe the set of permissible types. Predefined ones include `any` (all types), `comparable` (types that support `==`), and `cmp.Ordered` (`<` and friends). You can write your own:

```go
type Number interface {
    ~int | ~int64 | ~float64 // ~ permits defined types with underlying type int, etc.
}

func Sum[T Number](vals []T) T {
    var total T
    for _, v := range vals {
        total += v
    }
    return total
}
```

The tilde `~` allows the constraint to match named types whose underlying type is one of the listed ones—essential for working with domain types like `type Celsius float64`.

---

### 2. Under the Hood

Go does not perform full monomorphisation like C++ or Rust. Instead, the compiler uses **GC shape stenciling**. Two types share the same *GC shape* if they have the same underlying pointer/non-pointer structure and are the same size. For all types of the same shape, the compiler emits a single implementation that receives a runtime dictionary holding type-specific information (method tables, equality functions, etc.). If types differ in shape, the compiler generates separate specialisations.

This hybrid approach balances three forces:

- **Compilation speed** – Generating one shape per group is cheaper than generating per concrete type.
- **Code size** – Avoids the binary bloat of full template expansion.
- **Performance** – The dictionary is passed as an implicit argument; indirect calls through it are inevitable for interface-like constraints, but for concrete constraints such as `int | float64` the compiler can often inline the operator directly after specialisation.

For a generic function `Max[T cmp.Ordered]`, the compiler emits one shape for all pointer‑sized types with the same GC shape (e.g., `int`, `float64` share a shape on 64‑bit machines) and a separate one for strings (which contain a pointer). The generated code carries an implicit dictionary parameter that provides the comparison function.

**Implications:**

- There is no run‑time type creation. Generic code is fully resolved at compile time.
- The dictionary mechanism adds a thin indirection only when the constraint relies on an interface method. For constraints listing concrete types, the dictionary can often be elided.
- Type assertions on a value of a generic type are legal only if the constraint allows them; e.g., a type switch on a value of type `T` constrained by `any` is valid, but the compiler emits a runtime type check.

The **Go 1.18 spec** prohibits generic methods on a non‑generic type, and a method on a generic type cannot introduce additional type parameters. This limits the complexity of the dictionary and keeps the implementation tractable.

---

### 3. Why This Design?

The question “Why did Go wait so long?” has a single answer: every generics design was evaluated against the **three pillars**—simplicity, fast compilation, and orthogonality with existing interfaces. The accepted design (the “contracts” proposal evolved into interface constraints) satisfied the core team because:

1. **It feels like Go.** Type parameters look like an extension of the interface system you already know. Constraints are just interfaces with a union of types. No new kind of abstraction was invented.

2. **No overloading, no specialization.** Go refuses to let generics become a parallel meta‑language. You cannot partially specialise a generic function for a particular type—there is only one body. This forces you to keep generic code simple and predictable.

3. **Type inference reduces ceremony.** Explicit type arguments are almost never required. The compiler infers them from the function arguments, preserving Go’s declarative readability.

4. **Interface satisfaction remains structural.** A type satisfies a constraint if it meets the constraint’s terms—there is no “implements” keyword. The implicit satisfaction culture is preserved.

The alternative designs that were rejected include:
- **C++‑style templates** – full monomorphisation, header‑only code, slow builds.
- **Java‑style erasure** – run‑time type information erased, confusing with primitive types.
- **Rust‑style trait bounds** – powerful, but bring associated types, where clauses, and a complexity budget Go did not want.

The final design is deliberately less powerful: you cannot express “a type that has a `Compare` method *and* is comparable” in a single interface unless you write a constraint listing both interface and `comparable`. The team considered that acceptable for the 20 % of cases where generics truly help.

---

### 4. Competing Approaches

| Language | Mechanism | Code Bloat | Run‑time Cost | Constraints |
|----------|-----------|-------------|---------------|--------------|
| **C++** | Templates – full monomorphisation at instantiation | High (separate copy per type) | Zero (direct calls) | Implicit duck typing, Concepts since C++20 |
| **Java** | Type erasure – generic type replaced by upper bound at bytecode | Low (single erasure copy) | Virtual calls, boxing for primitives | Bounded type parameters (`extends`) |
| **Rust** | Monomorphisation with trait bounds | Medium‑high (per type copy) | Zero‑cost abstractions | Traits with associated types, where clauses |
| **Go** | GC shape stenciling + dictionaries | Low‑medium | Thin indirection for interface methods | Interface constraints with union elements |

**C++**: Templates are compile‑time duck typing. You can write `a + b` and if it compiles for a type, it works. This leads to cryptic error messages and long build times. Concepts (C++20) tame this, but the meta‑programming culture remains.

**Java**: Generics are a compile‑time illusion. At run time a `List<Integer>` and `List<String>` are the same `List`. Primitives must be boxed. The `? extends` and `? super` wildcards are a frequent source of confusion.

**Rust**: Generics are a first‑class abstraction tool. Monomorphisation ensures no performance overhead, but compile times suffer. Trait bounds with where clauses can express intricate relationships, at the cost of a steep learning curve.

**Go’s position**: It sits closer to Rust in spirit (compile‑time resolution) but with a drastically simpler constraint model and a conscious acceptance of a small runtime indirection to protect build speed and binary size.

---

### 5. Common Mistakes

#### 5.1 Reaching for generics when an interface suffices

A function that only calls methods on a value rarely needs type parameters. Write an interface parameter instead.

```go
// Over‑genericised
func PrintAll[T io.Writer](w T, msgs ...string) { … }

// Idiomatic
func PrintAll(w io.Writer, msgs ...string) { … }
```

#### 5.2 Using `any` as a constraint without type switching

A generic function constrained by `any` can do almost nothing with a value except assign it. You inevitably resort to reflection. If you find yourself reaching for a type switch, the function probably should not be generic.

#### 5.3 Ignoring the `comparable` constraint for map keys

The map key constraint is special: only `comparable` satisfies it. A plain `any` will not compile as a map key. Use `[K comparable, V any]`.

#### 5.4 Pointer‑value confusion with type parameters

```go
func (s *Set[T]) Add(v T) { … } // T is a value type; if T is already a pointer, you get double indirection
```

When T is itself a pointer type (e.g., `*User`), the method set becomes messy. Avoid parameterising on pointer types unless absolutely necessary; parameterise on the value type and let the caller use pointers explicitly.

#### 5.5 Assuming type parameters are interfaces

A constrained type parameter is not an interface value; it is a concrete type known at compile time. You cannot store values of different types in the same generic container unless the type parameter is instantiated with an interface type.

#### 5.6 Over‑constraining with union elements

```go
type StringOrBytes interface { ~string | ~[]byte }
```
You cannot call `len` on a value of type `T StringOrBytes` because `len` is not a method. Use separate concrete code paths or accept a `string` and provide a conversion helper.

---

### 6. Performance Considerations

The performance of generic code depends on **shape grouping** and **inlining**.

- **Monomorphised fast path**: When a constraint is a union of concrete types (`int | float64`), the compiler can inline arithmetic operators directly. Benchmarking shows that `Max[T cmp.Ordered]` often compiles to code indistinguishable from a hand‑written `int` version.

- **Interface‑bound slow path**: If the constraint contains an interface type, calls through the dictionary add a small overhead (roughly equivalent to a single interface method call). For most applications this is negligible.

- **Code size**: The GC shape approach prevents exponential code growth. A typical microservice sees a binary size increase of 2–5% after introducing generics, compared to 15–30% with C++ template heavy code.

- **Heap allocations**: Generic containers like `Set[T]` or `SliceMap[K, V]` allocate exactly what their concrete counterparts would. There is no implicit boxing; `T` is stored inline inside the struct or slice.

**Watch out for:**

- Passing large value types by value inside a generic function—it can trigger excessive copying. Use a pointer receiver if T may be large.
- When a generic function calls a method defined on the constraint’s interface, the compiler cannot inline across the dictionary boundary, preventing further optimisation.

---

### 7. Best Practices

1. **Start concrete, then abstract.** Write the function for the type you need today. Only introduce a type parameter when you encounter the exact same logic for a second type.

2. **Write the constraint before the body.** A well‑named constraint documents the contract. Keep constraints minimal; two methods and a union are almost always enough.

3. **Prefer functions over generic types.** A generic function like `Sort[T cmp.Ordered]` is easier to reason about than a `SortableSlice[T]` type with methods.

4. **Use the `~` prefix.** Unless you explicitly want to restrict to a single pre‑defined type, include `~` to allow idiomatic named types.

5. **Avoid type parameters on methods of non‑generic types.** You can’t do it anyway, but when you need generic behaviour on a method, consider making the whole type generic or using a top‑level function.

6. **Leverage type inference.** Callers should almost never write `Max[int](a, b)`. Trust inference; if inference fails, the code is probably too complex.

7. **Do not create “generic standard libraries.”** Utility packages full of generic `Map`, `Filter`, `Reduce` replicate the patterns of functional languages. Go slices and a simple `for` loop are often clearer and perform better.

---

### 8. Examples

#### 8.1 A reusable binary search

```go
// BinarySearch returns the index of target in a sorted slice, or -1.
func BinarySearch[T cmp.Ordered](s []T, target T) int {
    lo, hi := 0, len(s)-1
    for lo <= hi {
        mid := lo + (hi-lo)/2
        if s[mid] < target {
            lo = mid + 1
        } else if s[mid] > target {
            hi = mid - 1
        } else {
            return mid
        }
    }
    return -1
}
```

#### 8.2 A thread‑safe cache with `sync.RWMutex`

```go
type Cache[K comparable, V any] struct {
    mu sync.RWMutex
    m  map[K]V
}

func NewCache[K comparable, V any]() *Cache[K, V] {
    return &Cache[K, V]{m: make(map[K]V)}
}

func (c *Cache[K, V]) Get(key K) (V, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    v, ok := c.m[key]
    return v, ok
}

func (c *Cache[K, V]) Set(key K, val V) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.m[key] = val
}
```

#### 8.3 A pipeline stage over channels

```go
// Transform applies fn to each value from in and sends results to out.
func Transform[T, U any](ctx context.Context, in <-chan T, fn func(T) U) <-chan U {
    out := make(chan U)
    go func() {
        defer close(out)
        for {
            select {
            case <-ctx.Done():
                return
            case v, ok := <-in:
                if !ok {
                    return
                }
                select {
                case out <- fn(v):
                case <-ctx.Done():
                    return
                }
            }
        }
    }()
    return out
}
```

These examples demonstrate that generics shine when the **algorithm is independent of the type’s representation** but dependent on a minimal contract (ordering, comparability, or an interface).

---

### 9. Summary & Exercises

**Summary**

- Generics in Go use type parameters and interface‑based constraints to write code that operates on multiple types without sacrificing compile‑time safety.
- The implementation uses GC shape stenciling, a middle ground that keeps compilation fast and binaries lean while adding only a thin runtime indirection.
- The design philosophy stays true to Go’s values: simplicity, implicit satisfaction, and a clear separation between compile‑time and run‑time abstraction.
- Generics are **not** a replacement for interfaces; they are a complement for cases where the algorithm structure is identical but the types differ.

**Exercises**

1. **Generic Priority Queue**
   Implement a generic priority queue using `container/heap` (or your own heap) that works with any `cmp.Ordered` type. The API should be `pq.Push(v T)` and `pq.Pop() (T, bool)`. Make sure it is safe for concurrent use.

2. **Graph with generic nodes**
   Define a `Graph[Node any, EdgeWeight cmp.Ordered]` type where nodes carry a value of any type, and edges have a weight. Implement Dijkstra’s algorithm that returns the shortest path from a source node. The constraint on `Node` must be `comparable` for map lookups, and the edge weight must support addition and comparison. Write benchmarks comparing a concrete `int`‑weighted graph with the generic version.

3. **Generic retry helper**
   Write a function `Retry[T any](ctx context.Context, maxAttempts int, fn func(context.Context) (T, error)) (T, error)` that retries a call with exponential backoff. Use the `context` for cancellation. The function must be generic over the return value. Consider: should the constraint be `any`, or something else? Document the trade‑offs.
