## Chapter 10: Structs & Composition

If you’ve come from Java, C#, or C++, you’re used to reaching for classes, inheritance, and virtual methods. Go has none of those. Instead, it gives you **structs** and **composition**—and that’s a deliberate, powerful choice. This chapter isn’t about “how to mimic OOP in Go”; it’s about embracing a different mental model that leads to simpler, more maintainable systems.

**The “Aha!” moment:** Embedding looks like inheritance, but it’s not. It’s delegation and syntactic sugar. Once you stop trying to build type hierarchies and start composing behaviours, you’ll write code that’s easier to reason about, test, and change.

---

### 1. Basic Usage

A struct is a collection of fields. You define it, instantiate it, and access its fields.

```go
package main

import (
    "encoding/json"
    "fmt"
    "time"
)

// Basic struct definition
type User struct {
    ID        int
    Username  string
    Active    bool
    CreatedAt time.Time
}

// Struct with tags (for serialization, validation, etc.)
type Product struct {
    SKU   string  `json:"sku"`
    Price float64 `json:"price"`
    // Omitempty excludes zero-value fields from JSON
    InStock bool `json:"in_stock,omitempty"`
}

func main() {
    // Various ways to create structs
    var u1 User                     // zero-valued: ID=0, Username="", Active=false
    u2 := User{ID: 42, Username: "alice", Active: true}
    u3 := User{ID: 7, Username: "bob"} // CreatedAt gets zero time, Active false
    u4 := &User{ID: 99, Username: "charlie"} // pointer to struct

    u1.Username = "eve"
    fmt.Println(u1.Username) // "eve"

    // Struct tags in action
    p := Product{SKU: "X100", Price: 29.99, InStock: false}
    data, _ := json.Marshal(p)
    fmt.Println(string(data)) // {"sku":"X100","price":29.99}  (InStock omitted)

    // Anonymous structs – useful for tests or one-off shapes
    point := struct{ X, Y int }{10, 20}
    fmt.Println(point.X)
}
```

**Embedding (the “pseudo-inheritance”):**

```go
type Address struct {
    Street, City string
}

type Employee struct {
    Name    string
    Address // embedded field (no name, just type)
    Salary  int
}

func main() {
    e := Employee{
        Name:    "Diana",
        Address: Address{Street: "1 Go Way", City: "Cambridge"},
        Salary:  120000,
    }
    // Promoted fields
    fmt.Println(e.Street) // "1 Go Way" – directly accessible
    fmt.Println(e.City)   // "Cambridge"
}
```

---

### 2. Under the Hood

#### Memory Layout and Padding

Struct fields are laid out in memory **in the order you declare them**. The compiler adds padding to satisfy alignment requirements of the CPU architecture. Misordered fields can waste memory.

```go
type Bad struct {
    a bool   // 1 byte
    b int64  // 8 bytes – forces 7 bytes padding after a
    c bool   // 1 byte – followed by 7 bytes padding at the end
}
// Size: 24 bytes (on 64‑bit) due to alignment

type Good struct {
    b int64  // 8 bytes
    a bool   // 1 byte
    c bool   // 1 byte – then 6 bytes padding total?
}
// Size: 16 bytes (on 64‑bit)
```

Use `unsafe.Sizeof` and `unsafe.Alignof` to inspect, but the rule: **order fields from largest alignment to smallest** to minimise padding.

#### How Embedding Really Works

When you embed a type, the compiler **promotes its fields and methods** to the outer struct. No virtual table, no dynamic dispatch unless interfaces are involved. The embedded value is still a field with a synthetic name (the type name). You can access it explicitly:

```go
e.Address.Street == e.Street // true – both compile to the same thing
```

Promotion is shallow: if `Address` itself embeds `CityDetails`, those fields are **not** promoted to `Employee`. You’d need `e.Address.CityDetails.Zip`.

#### Method Promotion and Name Resolution

If the outer struct defines a method with the same name as an embedded method, the outer method **shadows** the embedded one – no overriding, no `super` call.

```go
type Log struct{}

func (Log) Info() { fmt.Println("Log.Info") }

type Service struct {
    Log
}

func (Service) Info() { fmt.Println("Service.Info") }

func main() {
    s := Service{}
    s.Info()          // "Service.Info" – outer wins
    s.Log.Info()      // "Log.Info" – explicit access
}
```

#### Zero Values

Every struct can be zero-valued. All its fields are recursively zero-valued. This is **intentional**: you never get `null` reference errors (unless you use pointers or interfaces). An uninitialized struct is ready to use.

```go
var config struct {
    Timeout time.Duration
    Retries int
}
// config.Timeout == 0, config.Retries == 0 – perfectly valid
```

---

### 3. Why This Design?

Go’s designers (Pike, Thompson, Griesemer) rejected classical inheritance for three core reasons:

1. **The Fragile Base Class Problem**  
   In languages like Java or C++, modifying a base class can silently break all derived classes. Go’s composition—explicit forwarding and embedding without overriding—eliminates that. Changes to an embedded type only affect the outer type if the field/method signature changes; there’s no unexpected virtual dispatch.

2. **Complexity of Hierarchies**  
   Deep inheritance trees are hard to understand and refactor. Go forces you to think in terms of **what a type does** (interfaces) and **what it has** (struct fields). This aligns with “prefer composition over inheritance”, a decades-old OOP best practice that languages like Java make arduous.

3. **Simplicity over “Reusability”**  
   Inheritance is often sold as code reuse, but it couples subtypes to supertypes. Go’s embedding provides **delegation reuse** without subtyping. If `Employee` embeds `Address`, `Employee` is **not** an `Address`. You cannot pass `Employee` where `Address` is required. That’s a feature: it prevents accidental Liskov substitution violations.

**The trade-off:** You have to write more explicit forwarding methods sometimes. But that explicitness is valuable—it makes control flow obvious.

---

### 4. Competing Approaches

| Language | Mechanism | Philosophy |
|----------|-----------|-------------|
| **Java / C#** | Classes, single inheritance, interfaces | “Is‑a” hierarchies with virtual methods. Heavy tooling (reflection, proxies) tries to mitigate rigidity. |
| **C++** | Multiple inheritance, templates | “Power at any cost.” Diamond problem, virtual inheritance, complex object lifetimes. |
| **Rust** | Structs + traits + composition (no inheritance) | Similar to Go: composition over inheritance. Traits define shared behaviour, but they’re explicit (no implicit interface satisfaction). Deref coercion gives a form of “inheritance‑like” convenience, but controversial. |
| **Python** | Multiple inheritance, mixins, duck typing | Highly dynamic. MRO (method resolution order) can be mind‑bending. “We’re all consenting adults.” |
| **Go** | Structs + embedding + interfaces (implicit) | “Have” instead of “is‑a”. Embedding is syntactic sugar, not subtyping. Interfaces provide polymorphism. |

**Key difference with Rust:** Rust’s `Deref` trait allows automatic dereferencing, which can emulate inheritance but is considered an anti‑pattern by many. Go has no such feature—embedding only promotes fields/methods, it never turns `Employee` into `Address`.

**Key difference with Java:** Go interfaces are satisfied implicitly. A struct does not need to declare `implements Logger`. This reduces coupling and enables **consumer‑defined interfaces** (a theme in Chapter 14).

---

### 5. Common Mistakes

#### Mistake 1: Thinking Embedding = Inheritance

```go
type Animal struct{ Name string }
type Dog struct{ Animal }

func (a Animal) Speak() { fmt.Println("...") }

func main() {
    var d Dog
    d.Speak()           // works, but Dog is not an Animal
    var a Animal = d    // COMPILE ERROR: cannot use d (type Dog) as type Animal
}
```

**Fix:** Use interfaces for polymorphism, not embedding.

#### Mistake 2: Copying Structs That Contain Pointers or Slices

```go
type Cache struct {
    items []int
}

func (c Cache) Add(v int) {
    c.items = append(c.items, v) // works on a copy of the slice header
}

func main() {
    c := Cache{items: []int{1,2,3}}
    c.Add(4)
    fmt.Println(c.items) // [1,2,3] – the copy's slice header was modified
}
```

**Fix:** Use pointer receivers for methods that mutate struct fields, or return the new struct (value semantics).

#### Mistake 3: Name Conflicts in Embedded Fields

```go
type A struct{ X int }
type B struct{ X int }
type C struct {
    A
    B
}
func main() {
    c := C{A: A{1}, B: B{2}}
    // fmt.Println(c.X) // ambiguous – compile error
    fmt.Println(c.A.X) // OK
}
```

**Fix:** Avoid ambiguous embeddings. If necessary, access via the explicit field name.

#### Mistake 4: Assuming Promoted Methods Are “Overridable”

You cannot override an embedded method. If you want to customise behaviour, don’t embed – **compose with an interface field** that can be swapped (strategy pattern).

#### Mistake 5: Misusing Struct Tags

Struct tags are strings; misspelling a key (`json:"sku,omitempty"` vs `jsno:"sku"`) fails silently. Always use constants or define a test that marshals/unmarshals to verify.

---

### 6. Performance Considerations

**Pass by Value vs. by Pointer**

- Small structs (≤ 4 machine words, roughly 32 bytes on 64‑bit) are **cheap to copy** and can stay on the stack.
- Large structs (e.g., > 64 bytes) should usually be passed via pointer to avoid copying cost.
- But pointer passing adds heap allocation if escape analysis fails. Profile before optimising.

**Escape Analysis Example**

```go
type Point struct{ X, Y int }

func NewPoint() *Point {   // returns pointer
    p := Point{X: 10, Y: 20}
    return &p               // escapes to heap because reference leaves function
}
```

If you return the `Point` by value, it stays on stack.

**Memory Alignment Waste**

Poorly ordered fields can waste 30-50% memory in struct-heavy workloads. For high‑performance systems (caches, game engines, network parsers), order fields by alignment.

**Cost of Interface Conversion**

When you assign a struct to an interface, two words are allocated (type pointer + data pointer). If the struct fits in one machine word, it’s stored directly; otherwise it’s boxed (heap). Repeated conversions can cause GC pressure.

**Zero‑allocation Embedding**

Embedding does not increase allocation—it’s a compile‑time transformation. An `Employee` with an embedded `Address` occupies exactly the memory of its fields laid out contiguously (plus padding).

---

### 7. Best Practices

1. **Favour small structs.** Each struct should have a clear, single responsibility.

2. **Use value receivers for immutable behaviour, pointer receivers for mutation.** This makes ownership semantics clear.

3. **Embedding is for delegation, not abstraction.** Ask: “Does the outer type genuinely **have** the inner type’s behaviour?” If you need polymorphism, define an interface and store it as a field.

4. **Avoid deep embedding chains (more than 1 level).** Shallow composition is easier to understand and change.

5. **Name promoted fields intentionally.** If you embed `sync.Mutex`, the `Lock`/`Unlock` methods become part of your struct’s API. That’s fine for private types, but think twice for public APIs.

6. **Use struct tags consistently** – JSON, YAML, DB, validation. Define a constant for tag keys if used in multiple places.

7. **Construct structs with constructor functions** when zero value isn’t sufficient:

```go
type Server struct {
    addr string
    port int
}

func NewServer(addr string, port int) *Server {
    if port < 1 || port > 65535 {
        port = 8080
    }
    return &Server{addr: addr, port: port}
}
```

8. **For optional fields, prefer zero‑value meaning “disabled”** rather than pointers. Only use `*T` when you need to distinguish between “zero” and “not provided”.

---

### 8. Examples

#### Example 1: Composing an `io.ReadWriter` from `io.Reader` and `io.Writer`

```go
type ReadWriter struct {
    io.Reader
    io.Writer
}

// No extra methods needed – Read and Write are promoted.
func main() {
    var rw ReadWriter
    rw.Reader = strings.NewReader("hello")
    rw.Writer = os.Stdout
    buf := make([]byte, 5)
    rw.Read(buf)  // delegated to strings.Reader
    rw.Write(buf) // delegated to os.Stdout
}
```

#### Example 2: Configuration with embedded defaults

```go
type DefaultConfig struct {
    Timeout time.Duration
    Retries int
}

type ProductionConfig struct {
    DefaultConfig // embed defaults
    Endpoint      string
}

func NewProductionConfig(endpoint string) ProductionConfig {
    return ProductionConfig{
        DefaultConfig: DefaultConfig{Timeout: 30 * time.Second, Retries: 3},
        Endpoint:      endpoint,
    }
}

func main() {
    cfg := NewProductionConfig("https://api.example.com")
    fmt.Println(cfg.Timeout) // 30s – promoted
}
```

#### Example 3: Struct tags for validation (using `go-playground/validator` – but keep it stdlib for now)

```go
type LoginRequest struct {
    Username string `json:"username" validate:"required,min=3"`
    Password string `json:"password" validate:"required,min=8"`
}
```

(Real validation requires external lib; the tag is the contract.)

---

### 9. Summary & Exercises

#### Summary

- Structs group data. Embedding promotes fields/methods syntactically, but it’s **not inheritance**.
- Go favours **composition over inheritance** to avoid fragile hierarchies and encourage explicit design.
- Memory layout matters – order fields to minimise padding.
- Zero values make structs safe to use without explicit constructors.
- Use interfaces for polymorphism, not embedding.

#### Exercises

**Exercise 1: Composable Logger**  
Design a `Logger` struct that can write to multiple destinations (e.g., file, stdout, in‑memory buffer). Use embedding or interface fields so that adding a new destination doesn’t change the logging logic. Implement a `MultiLogger` that composes several loggers and sends each log line to all of them.

**Exercise 2: Vehicle System without Inheritance**  
You’re building a fleet management system. You have `Car`, `Truck`, `Motorcycle`. Each has `MaxSpeed`, `FuelType`, and a method `CalculateMaintenanceCost()`. Instead of a `Vehicle` base class, use composition: define a `VehicleInfo` struct with shared fields and embed it. Then implement the `Maintainable` interface (with `CalculateMaintenanceCost`). Write a function that accepts `Maintainable` and processes a slice of mixed vehicle types.

**Exercise 3: HTTP Middleware Chain using Struct Composition**  
Build a small HTTP server where each middleware is a struct that embeds an `http.Handler` and adds behaviour (logging, auth, request ID). Compose them by nesting – e.g., `LoggingMiddleware{Next: AuthMiddleware{Next: finalHandler}}`. Then refactor to use a `Middleware` function type and a `Chain` struct. Compare both approaches. Which feels more idiomatic? Why?
