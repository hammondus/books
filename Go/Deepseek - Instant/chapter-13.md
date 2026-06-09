## Chapter 13: Interfaces & Implicit Implementation

Interfaces in Go flip the conventional object-oriented type system on its head. Instead of explicitly declaring that a type *implements* an interface (as in Java or C#), Go uses **implicit satisfaction**—if a type has the required methods, it automatically satisfies the interface. This design choice fundamentally changes how we build abstractions, decouple components, and test systems.

In this chapter, we’ll dissect what interfaces are, how they work under the hood, why the Go team chose implicit satisfaction, and how to wield them effectively without falling into common traps.

### Basic Usage

At its core, an interface is a set of method signatures. Any concrete type that implements *all* of those methods automatically satisfies the interface—no `implements` declaration required.

```go
package main

import "fmt"

// Define an interface.
type Speaker interface {
    Speak() string
}

// Concrete type: Dog. Note: no explicit "implements Speaker".
type Dog struct{ Name string }

func (d Dog) Speak() string {
    return fmt.Sprintf("%s says woof!", d.Name)
}

// Concrete type: Robot.
type Robot struct{ Model string }

func (r Robot) Speak() string {
    return fmt.Sprintf("%s says beep boop", r.Model)
}

// Function that accepts any Speaker.
func MakeItTalk(s Speaker) {
    fmt.Println(s.Speak())
}

func main() {
    d := Dog{Name: "Fido"}
    r := Robot{Model: "R2D2"}
    
    MakeItTalk(d) // Fido says woof!
    MakeItTalk(r) // R2D2 says beep boop
}
```

**The empty interface and type assertions**

Every type implements the empty interface `interface{}` (aliased to `any` in Go 1.18+). Use it when you truly don’t care about the type—like `fmt.Println` does. To recover the concrete type, use a **type assertion** or **type switch**.

```go
var v any = 42
i := v.(int)         // Assert that v holds an int, panics if wrong
fmt.Println(i + 1)   // 43

// Comma-ok idiom to avoid panic.
s, ok := v.(string)
if !ok {
    fmt.Println("not a string")
}

// Type switch.
switch t := v.(type) {
case int:
    fmt.Printf("int: %d\n", t)
case string:
    fmt.Printf("string: %s\n", t)
default:
    fmt.Printf("unexpected type %T\n", t)
}
```

### Under the Hood

An interface value in Go is a **two-word pair**: a pointer to an *itable* (interface table) and a pointer to the concrete data. The itable contains the type of the concrete value and a function pointer for each method in the interface.

```
[ interface value: { itable, data_ptr } ]
```

- **itable**: generated at compile time for each concrete type–interface pair. It stores the concrete type’s method addresses mapped to the interface’s method order.
- **data_ptr**: points to the actual value (allocated on heap if the value does not fit into a single machine word; small values may be stored inline via compiler optimisation).

When you call a method through the interface (e.g., `s.Speak()`), the runtime:
1. Loads the itable from the interface value.
2. Dereferences the appropriate function pointer from the itable.
3. Calls that function with the `data_ptr` as the receiver.

**Implicit satisfaction at compile time**

The compiler checks satisfaction **automatically**. Given an assignment `var s Speaker = Dog{}`, the compiler asks: “Does `Dog` have a `Speak() string` method?” If yes, it emits an itable for `Dog/Speaker`. No `implements` keyword ever appears in source.

**Nil interface vs. nil concrete value**

A subtle but critical point: an interface is `nil` only if *both* its itable and data pointer are nil.

```go
var s Speaker      // nil interface (itable = nil, data = nil)
var d *Dog = nil   // nil concrete pointer
s = d              // s is NOT nil! (itable = Dog/Speaker, data = nil)

if s == nil {
    fmt.Println("won't print")
}
if s.Speak() {     // This will panic! (*Dog receiver called with nil)
    // ...
}
```

### Why This Design?

Go’s implicit interfaces are a deliberate reaction against the rigidity of **explicit interface implementation** (as in Java, C#, or Swift). The Go team observed several problems with explicit declarations:

1. **Unnecessary coupling**: In Java, adding an interface to an existing type requires modifying the type’s source code. In Go, you can define an interface that describes *someone else’s* type without their permission—the type never needs to know about your interface.

2. **Interface explosion**: Explicit systems encourage creating interfaces “just in case” (e.g., every service gets an `IService`). Go’s implicit satisfaction flips the script: you define interfaces **at the point of use**, not at the point of implementation.

3. **Dependency direction**: Clean Architecture and Dependency Inversion dictate that higher-level modules should define the interfaces they need (e.g., `UserRepository`). Lower-level implementations (e.g., `PostgresUserStore`) then satisfy those interfaces implicitly. No import cycle and no boilerplate.

The “aha!” moment is this: **interfaces in Go are not a declaration of capability; they are a contract that the caller requires**. The callee doesn’t need to declare loyalty to that contract—it simply must be able to fulfill it. This reduces coupling to nearly zero.

### Competing Approaches

| Language | Mechanism | Trade-off |
|----------|-----------|------------|
| **Go** | Implicit satisfaction | Decoupling, flexibility; but can make it harder to find all implementors of an interface (solved by tooling like `gopls`). |
| **Java/C#** | Explicit `implements` / `:` | Discoverability (IDE shows all implementors) at cost of boilerplate and coupling. Every new interface requires changing implementing classes. |
| **Rust** | Trait impl blocks | Explicit but *orphan rules* prevent implementing a foreign trait on a foreign type (similar to Go’s implicit approach but with restrictions). Zero-cost abstraction via monomorphisation. |
| **Haskell** | Typeclasses | Implicit (instance declarations are separate from data types). Closest to Go’s philosophy, but with more expressive power (associated types, functional dependencies). |
| **C++** | Abstract base classes (virtual) | Explicit inheritance, multiple inheritance complexity. No separate interface concept. |

**Key insight**: Go trades **explicit discoverability** for **low coupling**. In a large Java codebase, you can `Ctrl+Click` on an interface and find all implementors. In Go, you rely on static analysis (e.g., `gopls`’s “Find implementations” feature) or convention (small interfaces in the consumer’s package).

### Common Mistakes

**1. Interface pollution (the “mock everything” anti-pattern)**

New Go developers often define an interface for every struct, mimicking Java’s `IService`/`ServiceImpl` pattern. This is unnecessary and harmful—it adds indirection without benefit.

```go
// Bad: interface defined before a concrete need exists.
type UserService interface {
    GetUser(id int) (*User, error)
    CreateUser(u *User) error
}

type userService struct { db *sql.DB }
func (s *userService) GetUser(id int) (*User, error) { /*...*/ }
```

Instead, define interfaces **where they are used** (e.g., in the HTTP handler that needs a `UserGetter`).

**2. Pointer receiver vs. value receiver mismatches**

If an interface method is defined with a pointer receiver, you cannot assign a value of that type to the interface—only a pointer.

```go
type Incrementer interface { Inc() }

type Counter int
func (c *Counter) Inc() { *c++ }  // Pointer receiver

var inc Incrementer
inc = Counter(0)   // Compile error: Counter does not implement Incrementer (Inc method has pointer receiver)
inc = &Counter(0)  // OK
```

**3. Storing nil concrete values in interfaces**

As shown earlier, assigning a nil pointer to an interface does **not** make the interface nil. This leads to bugs where you think you’ve returned nil but the caller sees a non-nil interface.

```go
func GetSpeaker() Speaker {
    var d *Dog = nil
    return d   // Returns a non-nil Speaker with nil data
}

func main() {
    s := GetSpeaker()
    if s == nil {
        fmt.Println("never prints")
    }
}
```

**4. Type assertion panics**

Using a single-value type assertion (`v.(int)`) will panic if the assertion fails. Always prefer the comma-ok form unless you are absolutely certain of the type.

**5. Breaking interface satisfaction accidentally**

Adding a method to a type that shares a name with an interface method but with a different signature breaks satisfaction silently. Example: adding `Close() error` to a type that previously satisfied `io.Closer` (which requires `Close() error`)—wait, that’s fine. But if you change the method name or parameter types, you break all callers. This is a trade-off of implicit satisfaction: the break is silent until compilation.

### Performance Considerations

**Indirect call cost**

Calling a method via an interface requires an indirect function call through the itable. This prevents inlining and costs roughly 5–10ns per call (compared to a direct call). In hot loops, this can matter.

```go
// Direct call (inlinable)
d.Speak()

// Interface call (indirect, not inlinable)
var s Speaker = d
s.Speak()
```

**Heap allocation**

When a concrete value is assigned to an interface, the compiler performs **escape analysis**. If the value’s lifetime exceeds the stack frame (or the compiler cannot prove it’s safe), the value is heap-allocated, and the interface’s data pointer points to that heap copy.

```go
func returnsInterface() Speaker {
    d := Dog{Name: "Fido"}  // May escape to heap
    return d                // d is boxed into interface, typically heap allocated
}
```

**Devirtualisation optimisation**

When the compiler can statically know the concrete type, it bypasses the interface call.

```go
func process(s Speaker) {
    s.Speak()  // Indirect call
}

func main() {
    d := Dog{}
    process(d) // Still indirect inside process
    d.Speak()  // Direct call, inlinable
}
```

**Zero-sized interfaces**

The empty interface `any` has the same two-word representation. Passing `any` does not avoid allocation if the value escapes.

**Best practice**: Accept interfaces, return structs. This pushes the boxing cost to the caller (who may already have the value on heap) rather than forcing allocation inside your function.

### Best Practices

**1. Small interfaces (the 1–2 method rule)**

The standard library is full of tiny interfaces: `io.Reader`, `io.Writer`, `fmt.Stringer`, `http.Handler`. Aim for the same. A large interface violates the principle of implicit satisfaction—fewer types will satisfy it.

**2. Consumer-defined interfaces**

Define interfaces in the package that *consumes* them, not in the package that provides the implementation. This inverts the dependency.

```go
// In package "http" (consumer)
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}

// In your application package (provider)
type myHandler struct{}
func (h myHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) { /*...*/ }
```

**3. Accept interfaces, return structs**

This gives callers flexibility while keeping your API concrete.

```go
// Good
func NewUserStore(db *sql.DB) *UserStore { ... }
func (s *UserStore) GetUser(ctx context.Context, id int) (*User, error) { ... }

// Bad (unless you have a good reason, like mocking for tests)
func NewUserStore(db *sql.DB) UserStoreInterface { ... }
```

**4. Use `any` sparingly**

`any` erases type information. Prefer concrete types or generic type parameters when you need type safety.

**5. Nil checks with interface values**

If a function returns an interface, never return a nil concrete value. Return the literal `nil` of the interface type.

```go
func GetSpeaker() Speaker {
    if noSpeaker {
        return nil   // OK, returns nil interface
    }
    return &Dog{}    // OK
}
```

### Examples

**Example 1: `io.Reader` usage**

```go
func CountBytes(r io.Reader) (int, error) {
    buf := make([]byte, 1024)
    total := 0
    for {
        n, err := r.Read(buf)
        total += n
        if err == io.EOF {
            return total, nil
        }
        if err != nil {
            return total, err
        }
    }
}

// Any type with Read(p []byte) (n int, err error) works
func main() {
    data := strings.NewReader("hello")
    count, _ := CountBytes(data)
    fmt.Println(count) // 5
}
```

**Example 2: Implementing `sort.Interface`**

```go
type Person struct {
    Name string
    Age  int
}

type ByAge []Person

func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }

func main() {
    people := []Person{{"Alice", 30}, {"Bob", 25}}
    sort.Sort(ByAge(people))
    fmt.Println(people) // Bob 25, Alice 30
}
```

**Example 3: Mocking with consumer-defined interface**

```go
// In package "user" (consumer)
type UserGetter interface {
    GetUser(ctx context.Context, id int) (*User, error)
}

type Handler struct {
    getter UserGetter
}

func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    user, err := h.getter.GetUser(r.Context(), 123)
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    fmt.Fprintf(w, "User: %+v", user)
}

// In production
type DBUserGetter struct { db *sql.DB }
func (g *DBUserGetter) GetUser(ctx context.Context, id int) (*User, error) { /*...*/ }

// In test
type mockGetter struct {
    user *User
    err  error
}
func (m mockGetter) GetUser(ctx context.Context, id int) (*User, error) {
    return m.user, m.err
}
```

### Summary & Exercises

**Summary**

- Go interfaces are satisfied **implicitly** – a type never declares “implements.” This reduces coupling and enables consumer-defined abstractions.
- An interface value is a two-word pair: (itable, data). The itable enables dynamic dispatch.
- Implicit satisfaction trades discoverability for decoupling, a deliberate design choice aligned with Go’s “less is more” philosophy.
- Common mistakes: interface pollution, mismatched receiver types, storing nil concrete values, and panic-prone type assertions.
- Performance: indirect calls (no inlining), potential heap allocation (boxing), but modern compilers devirtualise when possible.
- Best practices: small interfaces, define interfaces at the point of use, accept interfaces but return structs, return literal `nil` for interface-typed results.

**Exercises**

1. **Implement a thread-safe cache with pluggable eviction policies**  
   Define an interface `EvictionPolicy` with methods `OnAccess(key any)`, `OnInsert(key any)`, and `Evict() any`. Implement `LRU` and `FIFO` policies implicitly. Then build a `Cache` that stores `any` values but uses the policy. Write a test that verifies both policies behave correctly under concurrent access (use the race detector).

2. **Create a generic `Validator` using type assertions**  
   Write a function `Validate(input any, rules map[string]interface{}) error` that uses type switches and assertions to validate a struct’s fields. For example, `rules["Age"] = "min=18"`. This forces you to handle `any` and reflect on types. Then refactor it to use a small interface `FieldValidator` with methods like `ValidateField(fieldName string, value any) error`. Compare the two approaches.

3. **Build an HTTP middleware that recovers panics and logs via an interface**  
   Define a `Logger` interface with `LogError(msg string, err error)`. Implement it with `slog` (structured logging) and a no-op version for tests. Write a middleware that recovers panics from an `http.Handler`, logs the error via the injected logger, and returns a 500. Ensure your implementation never panics due to a nil concrete logger.
