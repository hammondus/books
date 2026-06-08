## Chapter 10. Structs & Composition

Structs are the backbone of data modeling in Go. They give you a lightweight, type-safe way to group related values, define explicit memory layouts, and attach behavior—without any of the baggage that comes with traditional class hierarchies. This chapter explores how structs work, what happens under the hood, why Go chose composition over inheritance, and how to wield structs in production systems.

---

### 1. Basic Usage

A struct declares a set of named fields, each with a type:

```go
type ServerConfig struct {
    Host    string
    Port    int
    UseTLS  bool
    Timeout time.Duration
}
```

You create instances with field-name literals (idiomatic) or positional initialization (rarely used, brittle):

```go
cfg := ServerConfig{
    Host:   "localhost",
    Port:   443,
    UseTLS: true,
}
```

Omitted fields take their **zero value**—a deliberate, useful default. `new(ServerConfig)` returns a pointer to a zero‑initialized struct; so does `&ServerConfig{}`.

#### Struct Tags

A unique Go feature: after a field’s type you can attach a raw string literal that acts as metadata for reflection-based libraries.

```go
type User struct {
    ID    int    `json:"id" db:"user_id"`
    Name  string `json:"name" validate:"required"`
    Email string `json:"email,omitempty"`
}
```

Tags follow the convention `key:"value"`, with spaces separating multiple key-value pairs. The standard library’s `encoding/json`, `encoding/xml`, and many third-party packages (ORMs, validators) rely on this mechanism. Tags are completely ignored by the compiler—they have no effect on type safety or memory layout.

#### Embedding (Anonymous Fields)

You can embed one struct (or interface) inside another by declaring a field with a type but no field name:

```go
type Base struct {
    ID   int
    Name string
}

type Extended struct {
    Base               // embedded struct
    ExtraField string
}
```

The `Extended` struct now *promotes* all fields and methods of `Base`. You can access `ext.Name` directly, as if it were a field of `Extended`. This is the cornerstone of Go’s composition model. If both `Extended` and `Base` have a field with the same name, the outer field shadows the inner one; you can still reach the embedded field via the explicit type name: `ext.Base.Name`.

---

### 2. Under the Hood

Understanding the memory layout of a struct is essential for writing cache‑friendly, allocation‑conscious code.

**Field Layout and Alignment**
The compiler lays out struct fields in declaration order, respecting the alignment requirements of each type. For example, on a 64‑bit system:

```go
type A struct {
    a byte   // 1 byte, offset 0
    b int64  // 8 bytes, offset 8 (7 bytes of padding)
    c byte   // 1 byte, offset 16
} // total size: 24 bytes (padded to multiple of 8)
```

You can reduce padding by reordering fields by descending size:

```go
type A struct {
    b int64
    a byte
    c byte
} // size: 16 bytes
```

The `unsafe.Sizeof` and `unsafe.Alignof` functions expose these details, though you rarely need them outside of low‑level optimization or serialization.

**Embedding’s Flat Layout**
Embedded fields are not separate allocations. The compiler *flattens* the embedded struct’s fields directly into the outer struct’s layout, exactly as if you had copied the fields by hand. Consequently, embedding adds zero memory overhead beyond the sum of the fields. The promoted field access is resolved at compile time by adjusting the base pointer—there is no runtime indirection.

**Zero Values**
A struct’s zero value is a struct where every field is set to its own zero value. This design avoids uninitialised‑memory bugs. A `sync.Mutex` can be used directly after declaration because its zero value is an unlocked mutex—no constructor required.

**Stack vs. Heap**
Structs are value types; passing them around creates a full copy. Small structs may stay on the stack, while taking the address of a local struct and returning it causes an escape to the heap. When a struct is stored inside an interface value, the data is typically copied and the interface holds a pointer to a heap‑allocated copy (or the data is small enough to be stored inline, depending on the implementation). Profiling with `go build -gcflags="-m"` reveals escape decisions.

**Tags in the Runtime**
Struct tags are stored as part of the type’s metadata in the runtime’s `structtype` representation. They are only accessible through the `reflect` package; the compiler does not interpret them. This keeps the language simple while giving libraries a powerful hook.

---

### 3. Why This Design?

Go deliberately avoids classes, inheritance, and all the associated complexity. The choice to build data modeling around structs and composition reflects three fundamental pillars:

1. **Data, Not Objects**
   A struct is a transparent collection of fields. Methods are functions with a receiver, not privileged members of a class hierarchy. This separation encourages you to model data as plain facts and attach behaviour where it makes sense. There is no “protected” or “private” inheritance, no super‑class constructor chains.

2. **Composition over Inheritance**
   The classic OOP deep hierarchy (e.g., `Animal` → `Mammal` → `Dog`) often becomes a maintenance nightmare. Go replaces that with explicit composition. Embedding gives you method promotion—syntactic sugar that lets you call an embedded type’s methods as if they belonged to the outer type—but it never creates an “is‑a” relationship. The outer struct contains the inner struct; it is **not** a subtype. This sidesteps the fragile base class problem, the diamond problem, and method‑override ambiguity.

3. **Simplicity for Tooling and Humans**
   Without classes, type hierarchies are flat. `gofmt`, `go vet`, and the reader’s brain can understand the complete set of fields and methods by looking at a single struct definition. Tags provide a declarative metadata mechanism without requiring the language to adopt annotations or attributes. The result is a codebase that is easier to refactor and reason about.

---

### 4. Competing Approaches

**Java / C#**
Java models behaviour and data together inside classes, with inheritance as the primary reuse mechanism. Go’s structs and interfaces invert this: data is plain, interfaces are satisfied implicitly, and code reuse happens through embedding and small interface contracts. Where Java leans on abstract classes and protected members to share code, Go says “just embed a concrete type or require an interface.” This often leads to flatter, more decoupled designs.

**C++**
C++ structs are classes with public default access; they support multiple inheritance, virtual functions, constructors/destructors, and operator overloading. Go intentionally strips all of that away. The Go philosophy argues that the power of C++ features comes at a high complexity cost—both in compilation speed and mental model. In Go, you get one mechanism for composition (embedding) and a clean separation of data and behaviour.

**Python**
Python’s dataclasses come close to Go structs, but Python supports multiple inheritance and mixins through its MRO. Go’s embedding is akin to single‑inheritance mixins: you embed a type and gain its methods, but you cannot embed two types that have conflicting method names without a compile‑time ambiguity error (which Go resolves by requiring explicit disambiguation). Python’s dynamic nature and lack of static type checking for field access place it in a very different design space.

**Rust**
Rust has structs (and enums) with no inheritance. Code reuse comes via traits with default method implementations and via manual delegation. Go’s embedding is more direct—you don’t need to write delegation boilerplate to expose a contained type’s methods. Rust’s trait system requires explicit implementation, which is more verbose but also more explicit. Go’s approach is simpler, though less expressive for advanced generic programming (pre‑generics; now generics help close some gaps).

**C**
C structs are purely data containers; you can simulate methods via function pointers stored in the struct. Go adds methods and embedding while retaining C’s memory‑layout transparency. This makes Go a natural evolution for systems programmers who want better type safety and code organisation without an OOP straitjacket.

---

### 5. Common Mistakes

**1. Accidentally Copying Large Structs**
Seasoned engineers coming from Java or C# (where everything is a reference) often forget that structs are value types. Passing a large struct by value to a function creates a full copy on the stack, potentially causing performance degradation and increased GC pressure if that copy escapes. **Fix:** use a pointer receiver or `*T` arguments.

```go
// Heavy copy:
func (b BigBuffer) Process() { … } // copies entire buffer

// Better:
func (b *BigBuffer) Process() { … }
```

**2. Shadowing Embedded Fields**
If the outer struct defines a field that shares a name with an embedded field, the outer field wins. The embedded field is still accessible via the embedded type’s name, but this can surprise readers:

```go
type Base struct { Value int }
type Outer struct {
    Base
    Value string // shadows Base.Value
}
o := Outer{}
o.Value = "shadow" // string field
o.Base.Value = 42  // explicit path needed
```

**3. Method Promotion Confusion with Pointers**
Method promotion rules differ based on whether you embed a value or a pointer:

- If you embed a type `T`, only the methods with value receivers on `T` are promoted.
- If you embed `*T`, methods with both value and pointer receivers are promoted (because a `*T` can call value‑receiver methods).

This matters when you want to promote a method that has a pointer receiver. Embedding a value will not promote that method:

```go
type Counter struct{ count int }
func (c *Counter) Increment() { c.count++ }

type Container struct { Counter } // embeds value
// c := Container{}; c.Increment() // compile error: Increment not promoted
```

You must embed `*Counter` to promote `Increment`.

**4. Treating Embedding as Inheritance**
Developers migrate from OOP and build deep embedding chains: `Base → Extended → MoreExtended`. This reintroduces the coupling and readability problems Go was designed to avoid. Prefer a flat structure where a type composes several small, independent pieces via named fields, reserving embedding for satisfying interfaces or when a truly transparent delegation is desired.

**5. Misusing Struct Tags**
Tags are strings; the compiler doesn’t validate their format. A typo like `jsno:"id"` will only be noticed at runtime when the serialization library ignores the field or fails silently. Use tools like `go vet` or linters that check standard tag keys.

**6. Exporting Unexported Fields Accidentally**
Embedding a type with unexported fields inside an exported struct does not magically export those fields. They remain inaccessible from outside the defining package, even through the outer struct. This is by design and protects encapsulation, but it can surprise newcomers.

---

### 6. Performance Considerations

**Struct Size and Padding**
Minimizing padding by reordering fields can shrink a struct from 24 to 16 bytes, reducing cache line pressure and memory bandwidth. This is especially important in hot‑path slices and maps where millions of instances exist.

**Copy Cost**
A `func(x MyStruct)` call copies the entire struct. For structs over ~64 bytes (a heuristic, not a rule), passing a pointer is usually cheaper, but that also forces a heap allocation if the value wasn’t already addressable. Measure with benchmarks; avoid premature optimization.

**Embedding Has Zero Runtime Overhead**
Embedding is entirely a compile‑time decoration. The resulting code is equivalent to hand‑written field access and method forwarding. No vtable dispatch or pointer dereference is added beyond what the underlying fields require.

**Interface Boxing of Structs**
When you assign a struct value to an interface variable, the runtime creates a copy of the struct and stores a pointer to it inside the interface. This involves a heap allocation unless the compiler can prove the interface value doesn’t escape. In performance‑critical code, consider passing interfaces that are already implemented by a pointer type (e.g., `*MyStruct`) to avoid the struct copy.

**Zero Value Efficiency**
Zero‑value structs are cheap to construct. The runtime often just zeros a stack region. Embrace the “zero value useful” pattern to avoid constructors.

**Tag Processing**
Reflection‑based tag parsing, like `encoding/json`, carries overhead per marshal/unmarshal operation. The upcoming `json/v2` in Go 1.23+ aims to reduce this by caching tag metadata, but for extreme throughput, you might write custom marshalers or use code generation.

---

### 7. Best Practices

**Field Naming and Initialization**
Always initialize structs using field names, not positional syntax. It documents the intent and survives field addition/deletion.

**Small, Cohesive Structs**
A struct should represent a single concept. If you find a struct with 15 fields serving multiple unrelated purposes, split it. The compiler will still produce efficient code, but the human cost is high.

**Prefer Named Fields over Embedding**
Embedding introduces a tight coupling: changing the embedded type’s exported surface affects all its users. Use a named field unless you explicitly want to “inherit” the interface and promote methods. A good rule of thumb: embed interfaces to indicate required behaviour (e.g., `type MyReader struct { io.Reader }`), embed concrete types only when that type is an integral part of the outer type’s identity and you intend to expose its entire API.

**Export with Care**
Because struct fields are accessed directly (without getters/setters), an exported field becomes part of your public API forever. For data‑transfer objects (configs, API responses) that’s fine. For types with invariants, keep fields unexported and provide exported methods that enforce rules.

**Zero‑Value‑Useful Structs**
Design types whose zero value is immediately usable. `sync.Mutex` and `bytes.Buffer` are canonical examples. This eliminates the need for constructors and reduces boilerplate.

**Tag Hygiene**
Use the standard `key:"value"` format. Multiple tags are space‑separated: `json:"name" xml:"name"`. For optional fields, use the `omitempty` convention. When using validation libraries, ensure tags are concise and documented.

**Composition With Interfaces**
A powerful pattern: embed an interface to decorate behaviour.

```go
type LoggingReader struct {
    io.Reader
    log *slog.Logger
}
func (lr *LoggingReader) Read(p []byte) (int, error) {
    n, err := lr.Reader.Read(p)
    lr.log.Info("read bytes", "n", n, "err", err)
    return n, err
}
```

This composes orthogonal concerns without an inheritance chain.

---

### 8. Examples

**Example 1: Struct Tags and JSON Round‑Trip**

```go
type Product struct {
    ID    int     `json:"id"`
    Name  string  `json:"name"`
    Price float64 `json:"price,omitempty"`
}

func main() {
    p := Product{ID: 1, Name: "Gopher Plush"}
    data, _ := json.Marshal(p)
    fmt.Println(string(data)) // {"id":1,"name":"Gopher Plush"}

    var p2 Product
    _ = json.Unmarshal(data, &p2)
    fmt.Printf("%+v\n", p2) // {ID:1 Name:Gopher Plush Price:0}
}
```

**Example 2: Embedding and Method Promotion**

```go
type Logger struct{}

func (l Logger) Log(msg string) { fmt.Println("LOG:", msg) }

type Service struct {
    Logger // promoted method Log
}

func main() {
    s := Service{}
    s.Log("service started") // directly via promotion
}
```

**Example 3: Resolving Shadowed Fields**

```go
type A struct{ Value int }
type B struct {
    A
    Value string
}
func main() {
    b := B{}
    b.Value = "outer"      // string field
    b.A.Value = 42         // embedded int field
    fmt.Println(b.Value, b.A.Value)
}
```

**Example 4: Embedding a Mutex – A Cautionary Tale**
Embedding `sync.Mutex` promotes `Lock` and `Unlock` to the outer type, which is usually undesirable because it exposes locking to external callers.

```go
type SafeCounter struct {
    sync.Mutex // oops, public Lock/Unlock
    count int
}
// Better: use a named field
type SafeCounter struct {
    mu    sync.Mutex
    count int
}
```

---

### 9. Summary & Exercises

Structs in Go are the primary means to organise data: they are plain, memory‑efficient, and deliberate in their lack of inheritance. Composition through embedding provides method promotion without creating fragile hierarchies, while struct tags elegantly decouple metadata from code. Understanding alignment, copying costs, and the nuances of method promotion is essential for building performant, idiomatic systems.

**Exercises**

1. **Thread‑Safe Configuration Cache**
   Design a generic `Cache[K comparable, V any]` struct that uses composition with a read‑write mutex (named field, not embedded) to protect a map. Provide methods `Get`, `Set`, and `Delete`. Then, write a benchmark that compares the throughput of your cache versus a simple `map` with external locking. Discuss why embedding the mutex would be a poor choice for a public API.

2. **Refactor a Java‑Style Hierarchy**
   Imagine a Java system with `AbstractVehicle`, `Car extends AbstractVehicle`, and `ElectricCar extends Car`. Each adds fields and overrides a `Drive()` method. Refactor this into Go using structs, interfaces (`Vehicle`), and explicit composition. Highlight where Go’s approach prevents the brittle base class problem and how you would handle shared behaviour (e.g., starting an engine) without inheritance.

3. **Build a Tag‑Driven Environment Mapper**
   Create a package that populates a struct from environment variables using struct tags. For example, a field `Port int` with tag `env:"PORT"` should be set to the value of the `PORT` environment variable (parsed as an integer). Support basic types (`string`, `int`, `bool`) and allow a `default:"..."` tag. Implement this with `reflect`, then discuss the trade‑offs (compile‑time safety, performance, error handling) versus generating code with `go:generate`.
