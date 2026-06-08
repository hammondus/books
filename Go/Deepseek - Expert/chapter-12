## Chapter 12. Methods & Receivers

### 1. Basic Usage

In Go, a **method** is a function that has a *receiver* — an extra parameter placed before the function name. It binds the function to a specific type. Methods can be defined on any named type (including structs) declared in the same package. The receiver can be a value (`T`) or a pointer to the type (`*T`).

```go
package main

import "fmt"

// Counter is a simple integer counter.
type Counter struct {
    n int
}

// Value is a method with a value receiver; it receives a copy of c.
func (c Counter) Value() int {
    return c.n
}

// Increment uses a pointer receiver; it can mutate the original counter.
func (c *Counter) Increment() {
    c.n++
}

func main() {
    var c Counter // zero value: n=0
    c.Increment() // Go automatically takes address: (&c).Increment()
    fmt.Println(c.Value()) // prints 1
}
```

You can also attach methods to non‑struct types.

```go
type ByteSize float64

func (b ByteSize) Kilobytes() float64 {
    return float64(b) / 1024
}
```

The compiler makes calling methods convenient: you can call a pointer‑receiver method on an addressable value, and a value‑receiver method on a pointer.

```go
c := Counter{}
c.Increment()           // ok, c is addressable, compiler inserts &
cp := &Counter{}
fmt.Println(cp.Value()) // ok, compiler dereferences cp: (*cp).Value()
```

Two restrictions exist:
1. You cannot call a pointer‑receiver method on an *unaddressable* value (for example, the return value of a function, or a map index expression).
2. You cannot define methods on a type from another package. The type must be declared locally.

Methods give Go a lightweight form of organization without classes. In the next sections we will unpack the runtime mechanics, the design rationale, and the practical trade‑offs.

---

### 2. Under the Hood

At the compiler level, a method is nothing more than an ordinary function with the receiver converted into a first argument. The transformation is straightforward:

```go
// Written by you:
func (c Counter) Value() int { return c.n }
// Compiled as (roughly):
func Counter.Value(c Counter) int { return c.n }

// Pointer receiver:
func (c *Counter) Increment() { c.n++ }
// Compiled as:
func (*Counter).Increment(c *Counter) { c.n++ }
```

This is why calling a method on a value like `c.Value()` is just syntactic sugar for `Counter.Value(c)`. The **method expression** syntax makes this explicit: `Counter.Value` is a function of type `func(Counter) int`. A **method value**, such as `c.Value`, is a closure that captures the receiver — by value if the receiver is `T`, or by pointer if it is `*T`.

```go
f := c.Value      // method value; f is func() int; captures copy of c
g := Counter.Value // method expression; g is func(Counter) int
fmt.Println(g(c)) // same as c.Value()
```

The compiler also automatically inserts the address‑of (`&`) or dereference (`*`) operators when you call a method. This works only when the value is **addressable** — that is, when the compiler can take its address safely (a variable, a struct field, a slice element, etc.). When a value is not addressable, you must explicitly assign it to a variable first. The compiler’s ability to insert `&` can cause a value to *escape* to the heap if the pointer receiver is used in a way that outlives the call. Escape analysis determines whether the receiver variable stays on the stack or moves to the heap.

#### Method Sets and Promotion

Every type has a **method set** that determines which methods it can call and which interfaces it satisfies.

- The method set of a type `T` consists of all methods with receiver `T`.
- The method set of `*T` includes both methods with receiver `T` and `*T` (promoted).

When a struct embeds another type, the methods of the embedded type are **promoted** to the outer struct. The compiler generates wrapper methods that forward the call to the embedded field. This is a compile‑time mechanism: there is no runtime dispatch or virtual table. Promotion works recursively.

```go
type A struct{}
func (A) Foo() {}

type B struct {
    A // embedded
}
// B now has a promoted method Foo with receiver B.
```

The method set of `B` includes `Foo`. Internally, the call `b.Foo()` is compiled to `b.A.Foo()`. The receiver is the embedded `A` value, not `B`. This has implications for interface satisfaction and shadowing: if `B` defines its own `Foo`, it shadows `A.Foo`, but `A.Foo` is still reachable as `b.A.Foo()`.

Understanding method sets is crucial because **interface satisfaction** is based entirely on method sets. If an interface requires a pointer‑receiver method, only a pointer to the type will satisfy it, not the value.

---

### 3. Why This Design?

Go’s method system is a deliberate departure from the class‑centric models of Java, C++, or C#. The guiding principle is **simplicity over complexity**, with composition winning over inheritance.

#### Separation of Data and Behavior
Methods are not defined inside a class block; they sit alongside the type declaration, anywhere in the package. This keeps the type definition minimal — just the data — and allows methods to be organized by functionality or grouped in separate files. It eliminates the notion of “class scope” and reduces the cognitive load of thinking about hierarchies.

#### Explicit Receiver
The receiver is declared explicitly, forcing the programmer to decide whether the method operates on a copy or the original. There is no implicit `this` or `self` that obscures mutability. This explicitness aligns with Go’s preference for clear, literal code. The receiver name is just a regular parameter; by convention it is a short, 1‑2 letter abbreviation of the type name (e.g., `c` for `Client`, `b` for `Buffer`). It’s never `this` or `self`.

#### No Monkey‑Patching
You cannot add methods to types defined in another package. This is a deliberate restriction. If you need to extend a foreign type, you use **composition** — wrap it in your own struct and delegate, or embed it to promote its methods. This keeps APIs stable and prevents surprising side effects. It also reinforces the “composition over inheritance” pillar.

#### Method Sets for Composition
Method sets and promotion turn embedding into a powerful yet simple tool for code reuse. By embedding a type, you gain its behaviour without the fragile base‑class problem. There is no virtual dispatch, no diamond‑of‑death, and no `super`. Shadowing a method is straightforward; you can still call the original via the embedded field. This design gently pushes developers toward composing small, focused types rather than building deep inheritance trees.

The **“Aha!” moment**: Methods in Go are not a new abstraction layer. They are plain functions with syntactic sugar. Once you internalize this, you see that a struct with methods is just a collection of data and the functions that operate on it — no magic, no hidden vtable, no implicit state. This mental model makes it obvious how to structure code for testability, clarity, and performance.

---

### 4. Competing Approaches

Comparing Go’s methods and receivers with other mainstream languages highlights the trade‑offs each environment makes.

**Java / C#**
Methods are bound inside class definitions. `this` is implicit and always refers to the current instance; mutability is not visible from the signature. Java methods are virtual by default (unless marked `final`), so every call incurs a vtable lookup. Go’s non‑interface methods are statically dispatched — no overhead. Java’s inheritance model encourages deep hierarchies; Go’s composition via embedding discourages them. Overloading is abundant in Java; Go has none, keeping the namespace flat and simple.

**Python**
Methods are functions that take `self` as the first parameter. This is conceptually close to Go’s explicit receiver, but Python treats all parameters as object references — there is no distinction between value and pointer semantics. Python allows dynamic addition of methods at runtime (monkey‑patching), whereas Go forbids it, preserving compile‑time safety and performance.

**C++**
Methods live inside class definitions. Const member functions (`const` after the signature) provide a similar contract to Go’s value receiver: they promise not to modify the object. However, C++ allows `mutable` fields to bypass this. C++ supports multiple inheritance and complex virtual dispatch; Go’s simplicity avoids that entire category of bugs. The receiver in Go is always explicit, making the call site clear.

**Rust**
Rust’s `impl` blocks and method syntax are the closest analogue. `&self` (immutable borrow) and `&mut self` (mutable borrow) map to Go’s value receiver (copy) and pointer receiver (mutability). Crucially, Rust’s borrow checker ensures safety without a GC, whereas Go uses a GC and often copies values. Rust’s approach guarantees no data races at compile time; Go’s is simpler but shifts that responsibility to the developer and the race detector.

**JavaScript**
Prototypal inheritance and dynamic `this` binding are worlds apart. Not directly comparable, but Go’s static typing and lack of dynamic dispatch outside interfaces make method calls deterministic and fast.

In summary: Go’s method system is unapologetically minimal. It sacrifices some expressiveness (no method overloading, no inheritance) for simplicity, predictability, and performance.

---

### 5. Common Mistakes

Seasoned engineers coming from class‑centric languages often stumble over the following nuances.

#### 1. Expecting a Value Receiver to Mutate
The most frequent error: writing a method with a value receiver intending to change the state. The mutation only affects the local copy; the caller’s value is untouched.

```go
func (c Counter) Increment() { c.n++ } // BUG: local copy modified
```

The fix is to use a pointer receiver. The compiler will not warn you because the code is perfectly valid — just likely not what you intended.

#### 2. Value Receiver on Large Structs
Every call with a value receiver copies the entire struct. For structs with many fields or large arrays, this can become a performance problem. A more subtle issue is structs containing a `sync.Mutex` or similar no‑copy type: copying them is illegal and `go vet` will flag it. This forces a pointer receiver.

```go
type SafeCounter struct {
    mu sync.Mutex
    n  int
}
// func (c SafeCounter) Increment() { … } // vet error: Lock passes lock by value
```

#### 3. Ignoring nil Receivers
A method with a pointer receiver can be called on a `nil` pointer. If the method tries to access fields without a nil check, it panics. Idiomatic Go often allows nil receivers when it makes the zero value useful.

```go
func (c *Counter) Value() int {
    if c == nil {
        return 0
    }
    return c.n
}
```

The standard library follows this pattern (e.g., `*bytes.Buffer`). Always decide whether nil is a meaningful receiver and document it.

#### 4. Method Set Mismatch with Interfaces
A method with a pointer receiver belongs only to the method set of `*T`, not `T`. Therefore a value of type `T` cannot satisfy an interface that requires that method. The error message is common:

```
T does not implement I (SomeMethod method has pointer receiver)
```

The fix is to use a pointer: `var i I = &t`. This is not a bug in the language; it’s a consequence of the explicit receiver choice. Always be mindful of how your types will be used in interfaces.

#### 5. Calling Pointer Methods on Unaddressable Values
A returned value from a function is not addressable:

```go
func NewCounter() Counter { return Counter{} }
NewCounter().Increment() // compiler error: cannot call pointer method on NewCounter()
```

Assign to a variable first, or change the factory to return a pointer.

#### 6. Mixing Receiver Types Inconsistently
The Go FAQ and Effective Go suggest that if any method of a type needs a pointer receiver, *all* methods should use pointer receivers for consistency. Exceptions exist (e.g., `time.Time` uses value receivers for non‑mutating operations), but in general, a uniform method set makes the type predictable for clients and avoids interface‑satisfaction confusion.

#### 7. Misunderstanding Method Shadowing in Embedding
When you embed a type and define a method with the same name, the outer method **shadows** the inner one. It does not override it. If you want to call the embedded method, you must use the explicit field name. This is often surprising to those expecting a `super` call or virtual dispatch.

```go
type Base struct{}
func (Base) F() {}

type Derived struct{ Base }
func (Derived) F() {} // shadows Base.F

d := Derived{}
d.F()      // calls Derived.F
d.Base.F() // calls Base.F
```

Keep shadowing in mind when designing composable APIs.

---

### 6. Performance Considerations

Receivers directly affect memory allocation, copying, and escape analysis.

#### Copy Cost
A value receiver copies the entire struct. For tiny structs (a few words), the copy is almost free and may be faster than a pointer dereference due to cache locality and the absence of indirection. For structs larger than, say, 64–128 bytes, the copy cost becomes noticeable in hot loops. Always benchmark.

#### Escape Analysis
When you use a pointer receiver, the compiler must guarantee that the receiver has a valid address. If the method call causes that pointer to escape the local stack (e.g., by returning it or storing it in a heap‑allocated structure), the receiver must be allocated on the heap. This adds GC pressure. Value receivers often stay entirely on the stack, avoiding allocations.

#### Inlining
The compiler can inline small methods with value receivers more readily because they don’t involve pointer operations. Inlined calls eliminate call overhead and often enable further optimisations. Pointer receiver methods can still be inlined if they are simple and the pointer does not escape, but the bar is higher.

#### Interface Dispatch
Methods called through an interface value (e.g., `var w io.Writer = &buf`) involve dynamic dispatch. That is a separate concern treated in Chapter 13. Direct method calls are static and cheap.

#### No‑Copy Types
If your struct contains a `sync.Mutex`, `sync.Cond`, or similar, a value receiver will copy the mutex — a runtime‑detected no‑copy violation. This is a hard correctness constraint, not just a performance one. Always use a pointer receiver in such cases.

#### Method Values and Closures
A method value like `f := obj.Method` captures the receiver. For a value receiver, the captured copy adds size to the closure; for a pointer receiver, only the pointer is captured. In hot loops, use a method expression to avoid repeatedly capturing the receiver.

**Guideline for receiver choice based on performance:**
- If the struct is small (≤ 64 bytes) and immutable‑like, value receiver.
- If the method needs to mutate or the struct is large, pointer receiver.
- If the struct embeds a `sync.Mutex` or similar, pointer receiver is mandatory.
- Measure with benchmarks; don’t guess.

---

### 7. Best Practices

**Receiver Naming**
Use a single letter or a short abbreviation that reflects the type. Examples:

- `func (c *Client) Do(req *Request)`
- `func (b *Buffer) Write(p []byte)`
- `func (t Time) Add(d Duration)`

Be consistent across all methods of the same type. Never use `this`, `self`, or generic names like `x`. The receiver is just another parameter; its brevity keeps the method body clean.

**Choosing Value vs Pointer Receivers**
- **Mutating methods:** pointer receiver.
- **Struct contains a sync primitive:** pointer receiver.
- **The type is large** (subjective, but if in doubt, pointer).
- **The type is a small value type** (like `time.Time`, `net.IP`, or a tiny config struct): value receiver.
- **Reference‑like types** (maps, channels, functions): value receiver works because they are already pointers under the hood. The copy is just a header.
- **Consistency:** If any method on the type requires a pointer receiver, it’s often best to make all methods pointer receivers, unless the type is explicitly designed to be used both ways and the non‑mutating ones stay value. Document your choice.

**Nil Receiver Handling**
If a pointer‑receiver method can be meaningfully called on `nil`, handle it gracefully. This makes the zero value of a pointer to your type useful and enables patterns like `var b *Buffer; b.Write(...)` (assuming `Write` handles nil). Not all types need this, but it’s a common idiom in the standard library.

**Method Grouping**
Keep methods close to the type definition, either in the same file or in a dedicated section. Avoid scattering them across many files without clear organization. A single type’s methods should be easy to find.

**Embedding and Promotion**
Use embedding to compose behaviours, not to simulate inheritance. When embedding, think about which methods you want to expose. You cannot unexport a promoted method, so if you embed a type with a wide interface, you expose all of it. Use a named field instead of embedding if you want control.

**Explicit Delegation**
If you override a promoted method by shadowing, but still need the original behaviour, call it explicitly: `w.EmbeddedType.Method()`. This is explicit, readable, and avoids confusion about which method runs.

**Avoid Method Overloading via Generics**
Go does not support method overloading, and that’s intentional. Don’t try to simulate it with generics unless there is a clear, idiomatic reason. Distinct method names are clearer.

**Document Mutability**
When a method uses a pointer receiver and modifies the state, make it obvious in the documentation. When a method returns a new value instead (like `time.Add`), use a value receiver to signal that the original is unchanged.

---

### 8. Examples

The following example pulls together method sets, embedding, interface satisfaction, and a common mistake in a single runnable program.

```go
package main

import (
    "fmt"
    "sync"
)

// Store is a thread‑safe key‑value store.
// It embeds sync.Mutex to promote Lock/Unlock.
type Store struct {
    sync.Mutex
    data map[string]string
}

// NewStore returns an initialised *Store.
func NewStore() *Store {
    return &Store{
        data: make(map[string]string),
    }
}

// Get retrieves a value safely.
// Pointer receiver because we lock the mutex.
func (s *Store) Get(key string) (string, bool) {
    s.Lock()
    defer s.Unlock()
    v, ok := s.data[key]
    return v, ok
}

// Set stores a value.
func (s *Store) Set(key, value string) {
    s.Lock()
    defer s.Unlock()
    s.data[key] = value
}

func main() {
    s := NewStore()
    s.Set("hello", "world")
    val, ok := s.Get("hello")
    fmt.Println(val, ok) // world true

    // Because *Store embeds sync.Mutex, it satisfies sync.Locker.
    var locker sync.Locker = s // ok: *Store has Lock/Unlock (pointer receivers)
    locker.Lock()
    // critical section
    locker.Unlock()

    // Common mistake: using a value instead of a pointer.
    // var s2 Store = *s
    // var l2 sync.Locker = s2 // compile error:
    //   Store does not implement sync.Locker (Lock method has pointer receiver)
    // Fix:
    // var l2 sync.Locker = &s2

    // Method expression: unbound function.
    get := (*Store).Get
    v, found := get(s, "hello")
    fmt.Println("via expression:", v, found) // via expression: world true
}
```

This example demonstrates:
- Pointer receivers for mutating operations and for satisfying `sync.Locker`.
- Method promotion through embedding a `sync.Mutex`.
- Interface satisfaction based on method set.
- The compile‑time error when using a value type where a pointer is required.
- Method expression syntax.

---

### 9. Summary & Exercises

**Summary**
Methods in Go are functions with a receiver, enabling behaviour to be associated with data without class hierarchies. Value receivers work on copies, pointer receivers allow mutation. Method sets and promotion turn composition into a first‑class design tool. The explicit receiver forces clarity about mutability and ownership, reinforcing the “composition over inheritance” and “share memory by communicating” philosophies. The compiler transforms methods into ordinary functions, so there is no hidden vtable or magic.

**Exercises**

1. **Thread‑Safe Cache with TTL**
   Design a generic cache using a `map`, a `sync.Mutex`, and a `time.Ticker` for background eviction. Decide on the receiver types for `Get`, `Set`, `Delete`, and a `Stats` method that returns hit/miss counters (use `sync/atomic`). Ensure that no method accidentally copies the mutex. Implement the cache and write a short test that exercises concurrent access.

2. **Composition for an HTTP Client Wrapper**
   Create a `Client` struct that embeds `*http.Client`. Add a `DoWithRetry` method that retries on transient errors, and a `Get` convenience method. Your wrapper should still satisfy the `http.RoundTripper` interface if the embedded client does. Handle nil receiver gracefully. Write code that uses your wrapper as a drop‑in replacement for `*http.Client` and discuss how method shadowing affects the call site.

3. **From Inheritance to Composition**
   Given a Java‑inspired design:
   - Base class `Animal` with `Speak()` and `Move()`.
   - Derived `Dog` and `Bird` with overridden behaviours.
   Refactor into Go using struct embedding and an `Animal` interface. Define a `BaseAnimal` struct with common fields, embed it in `Dog` and `Bird`, and implement the interface on each. Show that a slice of `Animal` works polymorphically. Discuss the trade‑offs: what do you gain and what do you lose compared to class inheritance? Pay special attention to method set rules and interface satisfaction.

These challenges force you to decide on receiver types, leverage embedding, and think about the “Go way” of structuring behaviour. Once you internalise that methods are just functions in disguise, you’ll find yourself writing cleaner, more composable Go code.
