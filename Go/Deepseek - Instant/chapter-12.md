# Chapter 12: Methods & Receivers

Go's approach to methods is deliberately minimalist: attach behavior to types without introducing classes, inheritance, or the complexity of traditional object-oriented programming. This chapter explores how methods work, the critical distinction between value and pointer receivers, and why Go's design forces clarity about mutation and memory.

---

## Basic Usage

Methods in Go are functions with a **receiver** argument declared between the `func` keyword and the method name. The receiver binds the method to a specific type.

### Value Receiver Syntax

```go
type Account struct {
    Balance float64
    Owner   string
}

// Value receiver: operates on a copy
func (a Account) Display() string {
    return fmt.Sprintf("%s: $%.2f", a.Owner, a.Balance)
}

func main() {
    acc := Account{Balance: 1000.50, Owner: "Alice"}
    fmt.Println(acc.Display()) // Alice: $1000.50
}
```

### Pointer Receiver Syntax

```go
// Pointer receiver: can modify the original
func (a *Account) Deposit(amount float64) {
    a.Balance += amount // Automatically dereferences a
}

// Also works with pointer receivers for consistency
func (a *Account) Withdraw(amount float64) error {
    if amount > a.Balance {
        return fmt.Errorf("insufficient funds")
    }
    a.Balance -= amount
    return nil
}

func main() {
    acc := &Account{Balance: 1000.50, Owner: "Alice"}
    acc.Deposit(500.75)
    // Balance is now 1501.25
}
```

### Method Calls on Values vs. Pointers

Go automatically handles conversions between values and pointers when calling methods:

```go
var accValue Account = Account{Balance: 100, Owner: "Bob"}
var accPtr *Account = &Account{Balance: 200, Owner: "Carol"}

// Value receiver works with both
accValue.Display() // fine
accPtr.Display()   // fine - Go automatically dereferences (*accPtr).Display()

// Pointer receiver works with both (but careful!)
accPtr.Deposit(50)   // fine - modifies accPtr
accValue.Deposit(50) // compiles but modifies a temporary copy! (see gotchas)
```

### Method Declarations on Any Type (Except Pointers and Interfaces)

You can declare methods on any type defined in your package, not just structs:

```go
type Celsius float64

func (c Celsius) ToFahrenheit() Fahrenheit {
    return Fahrenheit(c*9/5 + 32)
}

type Fahrenheit float64

func (f Fahrenheit) String() string {
    return fmt.Sprintf("%.1f°F", f)
}

type StringSlice []string

func (ss StringSlice) Join(sep string) string {
    return strings.Join(ss, sep)
}

func main() {
    temp := Celsius(25.0)
    fmt.Println(temp.ToFahrenheit()) // 77°F
}
```

---

## Under the Hood

### Method Set Rules

Every type has a **method set** — the collection of methods attached to it. The rules determine what methods can be called on values vs. pointers:

| Receiver Type | Method Set Includes |
|---------------|---------------------|
| `T` (value type) | All methods with receiver `T` |
| `*T` (pointer type) | All methods with receiver `T` AND all methods with receiver `*T` |

This asymmetry exists because a pointer can always be dereferenced to obtain a value, but the reverse is not always possible (taking the address of a value may not be allowed for temporary values).

### Memory Layout and Receiver Passing

When you call a method, the receiver is passed as the **first argument** to a function, following Go's calling convention:

```go
// Method declaration
func (a *Account) Deposit(amount float64)

// Compiled roughly to:
func Account_Deposit(a *Account, amount float64)
```

**Value receivers** copy the entire struct onto the stack (or heap if it escapes). For a struct with many fields, this copying has a cost.

**Pointer receivers** pass a single machine word (the pointer) regardless of struct size.

### Compiler Rewriting

The compiler automatically inserts dereferences and address-of operations when needed:

```go
var acc Account
acc.Deposit(100) // Compiler does: (&acc).Deposit(100)

var ptr *Account
ptr.Display() // Compiler does: (*ptr).Display()
```

This convenience hides the mechanical conversion but can lead to subtle bugs (see Common Mistakes).

### Method Values vs. Method Expressions

Go distinguishes between **method values** (bound to a specific receiver) and **method expressions** (unbound, treat method as a function):

```go
type Robot struct {
    Name string
    x, y int
}

func (r *Robot) Move(dx, dy int) {
    r.x += dx
    r.y += dy
}

// Method value: closes over the receiver
func exampleMethodValue() {
    r := &Robot{Name: "R2D2"}
    moveFn := r.Move // Method value - r is captured
    moveFn(5, 3)     // Moves r
    // Equivalent to: func(dx, dy int) { r.Move(dx, dy) }
}

// Method expression: treat method as a function with explicit receiver param
func exampleMethodExpression() {
    r := &Robot{Name: "C3PO"}
    moveFn := (*Robot).Move // Method expression
    moveFn(r, 10, -2)       // First argument is the receiver
}
```

Method expressions are useful for passing methods to higher-order functions or when you need to store the method without a receiver.

---

## Why This Design?

### No Classes, No Inheritance

Traditional OOP conflates three distinct concepts: data encapsulation, behavior attachment, and inheritance hierarchy. Go separates them:

- **Data encapsulation** happens at the package level (exported vs. unexported fields)
- **Behavior attachment** uses methods with receivers
- **Code reuse** uses composition and embedding, not inheritance

This design choice stems from Go's **simplicity-first** philosophy. Inheritance hierarchies inevitably become brittle and complex. A 2017 study of 1,000+ Java projects found that the average inheritance depth was 3.2 levels, but 15% of classes had depth >5, leading to "fragile base class" problems.

### Why Distinguish Value vs. Pointer Receivers?

Go forces you to **declare intent** about mutation. In languages like Java or C#, mutation is implicit through `this` references. In Python, methods always receive `self` as a reference.

Go's explicit receiver distinction accomplishes three goals:

1. **Clarity about mutation**: A value receiver signals "this method does not modify the receiver." A pointer receiver signals "this method may modify the receiver."

2. **Compiler optimizations**: The compiler can inline small value receiver methods without worrying about aliasing side effects.

3. **Immutability by convention**: You can design types that never use pointer receivers, making them effectively immutable (like `time.Time`).

### Why Methods on Any Type?

Go rejects the "everything is a class" model. Methods on primitive types and slices allow you to create **domain types** with domain-specific behavior:

```go
type UserID int64

func (uid UserID) String() string {
    return fmt.Sprintf("user_%d", uid)
}

type ConnectionPool map[string]*Connection

func (cp ConnectionPool) Get(addr string) (*Connection, error) {
    // method on a map type
}
```

This eliminates the need for "wrapper classes" or "utility classes" common in Java (`Collections`, `Objects`, `Math`). Behavior belongs with the data, but without ceremony.

---

## Competing Approaches

### Java/C#: Classes with Implicit `this`

```java
public class Account {
    private double balance;
    
    public void deposit(double amount) {
        this.balance += amount;  // 'this' is always a reference
    }
}
```

**Key differences:**
- Java methods are always reference-like (mutation is the default)
- Class-based dispatch with inheritance (`super`, `override`)
- Go has no implicit `this` — receiver is explicit and its type is declared

**Trade-off**: Java's implicit `this` is concise but obscures whether a method mutates or copies. Go's explicit receiver makes copying explicit, but requires more typing.

### C++: Value/Reference Semantics with `const`

```cpp
class Account {
    double balance;
public:
    void deposit(double amount) { balance += amount; }  // mutates
    void display() const { cout << balance; }           // const = read-only
};
```

C++ also distinguishes mutable and read-only methods (via `const`), but `const` applies to the implicit `this` pointer (`const Account* this`). Go's approach is more explicit: value receiver = copy semantics, pointer receiver = reference semantics.

**Trade-off**: C++ `const` is more flexible (can overload on constness) but more complex. Go trades flexibility for clarity.

### Rust: Explicit `self` with Ownership

```rust
impl Account {
    fn deposit(&mut self, amount: f64) {  // mutable reference
        self.balance += amount;
    }
    
    fn display(&self) -> String {  // immutable reference
        format!("{}", self.balance)
    }
}
```

Rust's approach is closest to Go's, but Rust distinguishes `self`, `&self`, and `&mut self` to express ownership. Go only distinguishes value vs. pointer (which maps roughly to `self` vs. `&mut self` in certain contexts).

**Trade-off**: Rust gives finer control over ownership and borrowing (e.g., consuming methods with `self`) but with steeper learning curve. Go's simplicity wins at the cost of some expressiveness.

### Python: Implicit `self`

```python
class Account:
    def deposit(self, amount):
        self.balance += amount  # 'self' is always the instance
```

Python's `self` is always a reference to the instance (like Java). There's no value receiver concept because Python's variables are always references. Go's value receivers are unique among popular languages — most languages either have only reference semantics (Java, Python, JS) or require explicit annotation (C++ `const`, Rust `&`).

**Why Go's approach is distinctive**: Go is one of the few modern languages where you can have a method that *cannot* modify the receiver because the receiver is a copy. This encourages **immutable design** without complex language features.

---

## Common Mistakes

### Mistake 1: Modifying a Value Receiver Expecting Persistence

```go
type Player struct {
    Health int
}

// WRONG: Value receiver - modifies a copy!
func (p Player) TakeDamage(amount int) {
    p.Health -= amount
}

func main() {
    player := Player{Health: 100}
    player.TakeDamage(30)
    fmt.Println(player.Health) // 100 - unchanged! Bug.
}
```

**Fix**: Use pointer receiver if mutation is needed:

```go
func (p *Player) TakeDamage(amount int) {
    p.Health -= amount
}
```

### Mistake 2: Calling Pointer Receiver Method on Temporary Values

```go
type Cache struct {
    data map[string]string
}

func (c *Cache) Set(key, value string) {
    if c.data == nil {
        c.data = make(map[string]string)
    }
    c.data[key] = value
}

func buildCache() Cache {
    c := Cache{}
    c.Set("key", "value") // Works because &c is taken implicitly
    return c
}

func main() {
    // This compiles but is dangerous:
    Cache{}.Set("key", "value") // Sets on a temporary that's immediately discarded!
    
    // More subtle: method call on function return
    getCache().Set("key", "value") // Sets on temporary if getCache returns by value
    
    // Correct way:
    c := Cache{}
    c.Set("key", "value")
}
```

### Mistake 3: Storing Method Values with Inconsistent Receivers

```go
type Counter struct {
    count int
}

func (c *Counter) Inc() { c.count++ }
func (c Counter) Value() int { return c.count }

func storeAndCall() {
    c := Counter{count: 0}
    
    incFn := c.Inc   // Method value with pointer receiver - captures &c
    valFn := c.Value // Method value with value receiver - captures copy of c
    
    incFn()
    incFn()
    
    fmt.Println(valFn()) // 0 - because valFn operates on the captured copy!
    fmt.Println(c.Value()) // 2 - the original changed
}
```

**Rule**: Method values capture the receiver at creation time with the semantics of the receiver type.

### Mistake 4: Nil Receiver Panics

```go
type Logger struct {
    prefix string
}

func (l *Logger) Log(msg string) {
    // PANICS if l is nil because l.prefix dereferences nil
    fmt.Printf("[%s] %s\n", l.prefix, msg)
}

func (l *Logger) SafeLog(msg string) {
    if l == nil {
        fmt.Printf("[nil] %s\n", msg)
        return
    }
    fmt.Printf("[%s] %s\n", l.prefix, msg)
}

func main() {
    var logger *Logger // nil
    logger.Log("hello") // PANIC!
    logger.SafeLog("hello") // works
}
```

**Best practice**: Methods that can receive nil should check explicitly. Otherwise, ensure callers never pass nil receivers.

### Mistake 5: Interface Satisfaction with Value Receivers Only

```go
type Greeter interface {
    Greet() string
}

type Person struct {
    Name string
}

// Value receiver only
func (p Person) Greet() string {
    return "Hello, " + p.Name
}

func main() {
    var g Greeter
    
    p := Person{Name: "Alice"}
    g = p   // OK: Person implements Greeter (value receiver)
    g = &p  // OK: *Person also implements Greeter
    
    // But if methods had MIXED receivers, pointer might be needed
}
```

The rule: If a type has any pointer receiver methods, the method set of the *value* type does NOT include those methods. This can lead to interface satisfaction surprises.

---

## Performance Considerations

### Receiver Copy Costs

Value receivers copy the entire struct. For large structs, this becomes expensive:

```go
type BigStruct struct {
    Data [1024]int // 8KB on 64-bit systems
    // ... more fields
}

// Value receiver copies 8KB+ on every call
func (b BigStruct) Process() {
    // ...
}

// Pointer receiver copies 8 bytes
func (b *BigStruct) ProcessFast() {
    // ...
}
```

**Benchmark example**:

```go
func BenchmarkValueReceiver(b *testing.B) {
    bs := BigStruct{}
    for i := 0; i < b.N; i++ {
        bs.Process()
    }
}
// Output: ~0.3 ns/op for pointer vs ~80 ns/op for value (8KB copy)
```

### Inlining Opportunities

The Go compiler can inline small methods. Value receivers have better inlining characteristics when they don't escape to heap:

```go
type Point struct{ X, Y int }

// Value receiver - likely inlined
func (p Point) Distance() int {
    return p.X*p.X + p.Y*p.Y
}

// Pointer receiver - less likely to inline if the compiler
// needs to prove pointer doesn't escape
func (p *Point) DistancePtr() int {
    return p.X*p.X + p.Y*p.Y
}
```

Check inlining with `go build -gcflags="-m"`:

```
./main.go:10:6: can inline Point.Distance
./main.go:16:6: cannot inline Point.DistancePtr: unhandled op INDREG
```

### Escape Analysis Impact

Pointer receivers can cause heap allocations when the compiler can't prove the pointer's lifetime:

```go
func NewAccount(owner string) *Account {
    a := Account{Owner: owner} // allocates on stack
    return &a                   // escapes to heap
}

// If a method with pointer receiver is called on a stack-allocated value,
// taking its address may cause an escape
```

### Method Call Overhead

Method calls have negligible overhead compared to function calls (the receiver is just another argument). However, **interface method calls** have indirect dispatch overhead:

```go
type Intf interface{ Do() }

type Impl struct{}

func (Impl) Do() {}

func directCall(i Impl) {
    for n := 0; n < 1_000_000; n++ {
        i.Do() // direct call - can be inlined
    }
}

func interfaceCall(i Intf) {
    for n := 0; n < 1_000_000; n++ {
        i.Do() // indirect call through itab - cannot inline
    }
}
```

**Rule**: Hot paths with millions of iterations should avoid interface dispatch when possible.

### Memory Optimization: Pointer Receivers for Large Types

```go
// Recommended: Use pointer receivers for any type > 64 bytes
type LargeBuffer struct {
    buf [4096]byte
    pos int
}

func (lb *LargeBuffer) Write(p []byte) (n int, err error) {
    // pointer receiver avoids 4KB copy
}
```

---

## Best Practices

### Consistent Receiver Types

**Choose either value or pointer receivers for all methods of a type** to avoid confusion:

```go
// GOOD: All pointer receivers
type User struct {
    ID   int64
    Name string
}

func (u *User) SetName(name string) { u.Name = name }
func (u *User) String() string       { return fmt.Sprintf("%d: %s", u.ID, u.Name) }

// ACCEPTABLE but rare: All value receivers (immutable type)
type Point3D struct{ X, Y, Z float64 }

func (p Point3D) Add(q Point3D) Point3D { return Point3D{p.X+q.X, p.Y+q.Y, p.Z+q.Z} }
func (p Point3D) Magnitude() float64    { return math.Sqrt(p.X*p.X + p.Y*p.Y + p.Z*p.Z) }

// BAD: Mixed receiver types without justification
type Confusing struct {
    data map[string]string
}

func (c Confusing) Read() string { return c.data["key"] }
func (c *Confusing) Write(v string) { c.data["key"] = v } // Inconsistent!
```

### Small Types (≤64 bytes) Without Mutation Can Use Value Receivers

```go
// Good candidate for value receiver
type Coordinate struct {
    Lat, Long float64
}

func (c Coordinate) DistanceTo(other Coordinate) float64 {
    // no mutation, small size
    return haversine(c.Lat, c.Long, other.Lat, other.Long)
}
```

### Nil-Safe Methods When Appropriate

Some types benefit from nil-safe methods, like `bytes.Buffer`:

```go
type SafeLogger struct {
    out io.Writer
}

func (l *SafeLogger) Log(msg string) {
    if l == nil {
        return // or log to default
    }
    fmt.Fprintln(l.out, msg)
}
```

### Getter/Setter Conventions

Go has no official getter/setter convention, but the community follows a pattern:

```go
type Customer struct {
    name string // unexported
    age  int
}

// Getter: omit "Get" prefix
func (c *Customer) Name() string {
    return c.name
}

// Setter: use "Set" prefix
func (c *Customer) SetName(name string) {
    if name == "" {
        panic("name cannot be empty")
    }
    c.name = name
}
```

### Use `T` Methods for Immutable Types Like `time.Time`

```go
// From standard library: time.Time uses value receivers exclusively
func (t Time) Add(d Duration) Time   // returns new Time
func (t Time) UTC() Time             // returns new Time
// No pointer receiver methods - Time is immutable by design
```

### Naming: Avoid Stuttering

```go
// BAD: Stuttering
type Player struct {}
func (p *Player) PlayerSave() {}   // "Player.PlayerSave"
func (p *Player) GetPlayerName() {} // "Player.GetPlayerName"

// GOOD: Concise
func (p *Player) Save() {}
func (p *Player) Name() string {}
```

---

## Examples

### Example 1: Builder Pattern with Pointer Receivers

```go
type ServerConfig struct {
    host    string
    port    int
    timeout time.Duration
    maxConn int
}

// Builder methods return the pointer for chaining
func (c *ServerConfig) SetHost(host string) *ServerConfig {
    c.host = host
    return c
}

func (c *ServerConfig) SetPort(port int) *ServerConfig {
    c.port = port
    return c
}

func (c *ServerConfig) SetTimeout(timeout time.Duration) *ServerConfig {
    c.timeout = timeout
    return c
}

func (c *ServerConfig) Build() (*Server, error) {
    if c.host == "" {
        c.host = "localhost"
    }
    if c.port == 0 {
        c.port = 8080
    }
    return &Server{
        host:    c.host,
        port:    c.port,
        timeout: c.timeout,
        maxConn: c.maxConn,
    }, nil
}

// Usage
func main() {
    cfg := &ServerConfig{}
    server, _ := cfg.SetHost("example.com").SetPort(443).SetTimeout(30*time.Second).Build()
    _ = server
}
```

### Example 2: Value Receiver for Immutable Vector Type

```go
type Vector struct {
    X, Y, Z float64
}

// All methods use value receivers - operations return new vectors
func (v Vector) Add(w Vector) Vector {
    return Vector{v.X + w.X, v.Y + w.Y, v.Z + w.Z}
}

func (v Vector) Sub(w Vector) Vector {
    return Vector{v.X - w.X, v.Y - w.Y, v.Z - w.Z}
}

func (v Vector) Dot(w Vector) float64 {
    return v.X*w.X + v.Y*w.Y + v.Z*w.Z
}

func (v Vector) Scale(s float64) Vector {
    return Vector{v.X * s, v.Y * s, v.Z * s}
}

func (v Vector) Magnitude() float64 {
    return math.Sqrt(v.Dot(v))
}

// No pointer receivers - Vector is immutable
```

### Example 3: Method Expression for Map/Filter Operations

```go
type Person struct {
    Name string
    Age  int
}

func (p Person) IsAdult() bool {
    return p.Age >= 18
}

func (p *Person) IncrementAge() {
    p.Age++
}

func filterPeople(people []Person, predicate func(Person) bool) []Person {
    result := make([]Person, 0, len(people))
    for _, p := range people {
        if predicate(p) {
            result = append(result, p)
        }
    }
    return result
}

func main() {
    people := []Person{
        {"Alice", 25},
        {"Bob", 17},
        {"Carol", 30},
    }
    
    // Method expression as predicate
    adults := filterPeople(people, Person.IsAdult)
    fmt.Printf("%d adults\n", len(adults)) // 2 adults
    
    // Method expression for mutation
    incAge := (*Person).IncrementAge
    for i := range people {
        incAge(&people[i])
    }
    // All ages increased by 1
}
```

---

## Summary & Exercises

### Key Takeaways

1. **Value receivers** operate on a copy; use them for small, immutable types or when methods don't need mutation.

2. **Pointer receivers** can modify the receiver; use them for mutation, large structs (>64 bytes), or when methods need to modify the original.

3. **Method sets** differ between `T` and `*T`: `*T` includes all methods, `T` only includes value receiver methods.

4. **Consistency matters**: Prefer all pointer receivers or all value receivers for a given type.

5. **Method values** capture the receiver at creation time and can lead to subtle bugs with mixed receiver types.

6. **Performance**: Value receivers copy data; pointer receivers add heap pressure. Benchmark to decide for hot paths.

### Exercises

#### Exercise 1: Design a Thread-Safe Counter with Proper Receivers

Create a `SafeCounter` type that wraps an `int64` and provides atomic increment, decrement, and read operations. Decide whether to use value or pointer receivers and justify your choice. Then implement:

- `Inc()` increments and returns the new value
- `Dec()` decrements and returns the new value
- `Value()` returns the current value
- `Reset()` sets the counter to zero

Write a race condition test using `go test -race` to verify correctness.

#### Exercise 2: Refactor Inheritance to Composition with Methods

Given the following Java-style inheritance:

```go
type Animal struct { Name string }
func (a *Animal) Speak() string { return "..." }

type Dog struct { Animal }
func (d *Dog) Speak() string { return "Woof" }

type Cat struct { Animal }
func (c *Cat) Speak() string { return "Meow" }
```

Refactor this to use composition without embedding. Create a `Speaker` interface and implement methods that show how Go's method design encourages composition over inheritance. Then, write a function `MakeSounds(speakers []Speaker)` that calls `Speak` on each and collects results.

#### Exercise 3: Identify and Fix Method Receiver Bugs

Analyze this broken cache implementation:

```go
type Cache struct {
    store map[string]string
}

func (c Cache) Set(key, value string) {
    if c.store == nil {
        c.store = make(map[string]string)
    }
    c.store[key] = value
}

func (c Cache) Get(key string) string {
    return c.store[key]
}

func NewCache() Cache {
    return Cache{}
}

func main() {
    c := NewCache()
    c.Set("a", "b")
    fmt.Println(c.Get("a")) // What prints? Why?
}
```

Fix the implementation to work correctly. Then, add a method `Len() int` that returns the number of items. Finally, discuss why the original implementation compiled without errors despite being broken.

#### Challenge: Implement a Method Expression Pipeline

Create a `Pipeline` type that stores a slice of operations. Each operation is a method expression that transforms a `string`. Implement:

```go
type StringProcessor struct {
    value string
}

func (sp *StringProcessor) Upper() { sp.value = strings.ToUpper(sp.value) }
func (sp *StringProcessor) Lower() { sp.value = strings.ToLower(sp.value) }
func (sp *StringProcessor) TrimSpace() { sp.value = strings.TrimSpace(sp.value) }
func (sp *StringProcessor) Reverse() { /* reverse sp.value */ }

type Pipeline struct {
    ops []func(*StringProcessor) // method expressions stored here
}

func (p *Pipeline) Add(op func(*StringProcessor)) { ... }
func (p *Pipeline) Run(initial string) string { ... }
```

Then create a pipeline that trims spaces, converts to upper, and reverses a string. Compare the performance of this approach vs. calling methods directly on a `StringProcessor` instance (benchmark). Explain when this pattern is useful.
