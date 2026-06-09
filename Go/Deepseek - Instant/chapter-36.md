## Chapter 36: Common Anti-Patterns

### 1. Basic Usage

Anti-patterns aren't syntax errors—they compile, run, and often work in small scales. The problem emerges when the codebase grows, when concurrency increases, or when a new engineer needs to understand what you wrote six months ago.

Here's a deceptively "correct" Go function that demonstrates multiple anti-patterns:

```go
// DON'T: Java-style abstraction layer
type UserRepository interface {
    GetUser(ctx context.Context, id string) (*User, error)
    SaveUser(ctx context.Context, user *User) error
}

type UserService interface {
    GetUser(ctx context.Context, id string) (*UserDTO, error)
    CreateUser(ctx context.Context, req *CreateUserRequest) error
}

type UserServiceImpl struct {
    repo UserRepository
    cache *redis.Client
    mapper *UserMapper
}

func (s *UserServiceImpl) GetUser(ctx context.Context, id string) (*UserDTO, error) {
    // Cached? Let's check.
    cached, err := s.cache.Get(ctx, "user:"+id).Result()
    if err == nil {
        var u User
        if err := json.Unmarshal([]byte(cached), &u); err == nil {
            return s.mapper.ToDTO(&u), nil
        }
    }
    
    u, err := s.repo.GetUser(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("failed to get user: %w", err)
    }
    
    // Async cache write (fire-and-forget - another anti-pattern)
    go func() {
        data, _ := json.Marshal(u)
        s.cache.Set(context.Background(), "user:"+id, data, time.Hour)
    }()
    
    return s.mapper.ToDTO(u), nil
}
```

The idiomatic Go version removes the unnecessary abstractions, eliminates the goroutine leak risk, and makes the caching behavior explicit:

```go
// DO: Direct, clear, minimal
type User struct {
    ID   string
    Name string
}

type UserStore interface {
    Get(ctx context.Context, id string) (*User, error)
    Save(ctx context.Context, u *User) error
}

type CachedUserStore struct {
    store UserStore
    cache *redis.Client
}

func (c *CachedUserStore) Get(ctx context.Context, id string) (*User, error) {
    cacheKey := "user:" + id
    
    // Synchronous cache with fallback
    if cached, err := c.cache.Get(ctx, cacheKey).Bytes(); err == nil {
        var u User
        if err := json.Unmarshal(cached, &u); err == nil {
            return &u, nil
        }
    }
    
    u, err := c.store.Get(ctx, id)
    if err != nil {
        return nil, err
    }
    
    // Write cache synchronously with context
    data, _ := json.Marshal(u)
    _ = c.cache.Set(ctx, cacheKey, data, time.Hour).Err() // Best-effort
    
    return u, nil
}
```

### 2. Under the Hood

Anti-patterns become costly because of how Go's runtime and compiler behave under pressure.

**Interface Indirection Cost:** Each interface method call requires a dynamic dispatch through an *itable* (interface table). While cheap (2-3 nanoseconds per call), the real cost is **inlining prevention**. The compiler cannot inline interface method calls, which eliminates a major optimization opportunity for small methods like getters or validation functions.

```go
// Compiler can inline this
func (u *User) IsActive() bool { return u.Status == "active" }

// Cannot inline (interface dispatch)
var activator interface{ IsActive() bool } = user
activator.IsActive() // Must go through itable
```

**Goroutine Overhead from Fire-and-Forget:** Launching a goroutine with `go func()` costs ~2KB for the stack (as of Go 1.21), plus scheduler overhead. The real problem is **orphaned goroutines**—if the parent context cancels or the program exits, those background goroutines continue running, preventing GC of their captured closures and potentially leaking memory indefinitely.

**Unbounded Channel Writes:** A write to an unbuffered channel blocks until a receiver is ready. In the "async cache write" anti-pattern above, if the cache consumer is slow or crashed, the goroutine blocks forever, leaking both the goroutine and the captured `User` object.

**Excessive Struct Tags:** Every struct tag is parsed at runtime by packages like `encoding/json`. While the parse result is cached, the memory overhead remains: each tag string is stored in the binary's read-only data section. For large-scale structs with multiple tags (`json:"field" xml:"field" db:"field" validate:"required"`), this adds measurable binary size and startup time.

### 3. Why This Design?

Go's designers intentionally omitted **inheritance**, **generics** (until 1.18), and **exceptions** to force simplicity. Anti-patterns often arise when developers try to re-implement patterns from other languages.

**Why no abstract classes?** Go's interface satisfaction is implicit. If you find yourself writing a "Base" struct with embedded fields and methods meant to be overridden, you're fighting the language. The Go philosophy says: *"Accept interfaces, return structs."* Embedding is for composition, not polymorphism.

**Why no constructors?** Go has `new(T)`, `&T{}`, and `make()`. The "constructor function" pattern (`func NewUser(...) *User`) is idiomatic, but returning an interface from a constructor is not—it breaks the `Accept interfaces, return structs` rule and forces heap allocation (escape analysis struggles with interface returns).

**Why were generics resisted for so long?** The Go team observed that most code doesn't need parametric polymorphism. The anti-pattern of "generic-ifying everything" emerged immediately after 1.18. The team's position remains: use code generation (`go:generate`) or `interface{}` (now `any`) for the 90% case, and reach for generics only when you have genuinely type-agnostic algorithms (slices, maps, channels, or math).

**Why no implicit this/self?** Go's explicit receiver parameter (`func (u *User) Save()`) makes it obvious whether the receiver is a value or pointer. Anti-patterns like mixing receiver types on the same type cause subtle bugs:

```go
type Counter struct { n int }
func (c Counter) Value() int { return c.n }      // value receiver
func (c *Counter) Inc() { c.n++ }                // pointer receiver

c := Counter{}
c.Inc()
fmt.Println(c.Value()) // 1? Wait, Inc worked on pointer, Value copied - fine, but confusing
```

### 4. Competing Approaches

| Language | Common Anti-Pattern | Go's Equivalent Anti-Pattern |
|----------|--------------------|------------------------------|
| **Java** | "AbstractFactoryFactoryBean" - deep inheritance hierarchies | Interface-heavy layering with single implementations; `UserService`, `UserServiceImpl`, `UserRepository`, `UserRepositoryImpl` |
| **Python** | Monkey-patching, dynamic attribute creation | Using `map[string]interface{}` (or `any`) instead of structs; reflection-based "magic" setters |
| **C++** | Deep template metaprogramming | Over-genericizing simple functions (e.g., `func Get[T any](m map[string]T, k string) T` - just write the loop) |
| **Rust** | Lifetime annotations everywhere | Fighting the GC by manual `sync.Pool` for every allocation (premature optimization) |
| **JavaScript** | Callback hell | Channel-of-channels patterns or nested `select` statements that should be pipelines |

**Specific comparison: Java's checked exceptions vs. Go's error handling.** Java's anti-pattern is `catch(Exception e) {}` - swallowing errors. Go's equivalent is `_, _ = doSomething()` - ignoring the error return entirely. Both silence failures, but Go's version is more explicit in its negligence.

**Comparison: C++ RAII vs. Go's defer.** C++ anti-pattern: manual resource management with `new`/`delete` leading to leaks. Go anti-pattern: forgetting `defer` on mutex unlocks or file closes, or worse, `defer` inside a loop (which stacks up until the function returns, not the iteration ends).

```go
// DANGER: defer in loop
for _, fname := range files {
    f, err := os.Open(fname)
    if err != nil {
        continue
    }
    defer f.Close() // Won't close until function returns - file handles accumulate!
    process(f)
}
```

### 5. Common Mistakes (The "Gotchas")

**Anti-pattern #1: Premature Interface Abstraction**

```go
// DON'T: Interface with single implementation
type StringFormatter interface {
    Format(s string) string
}

type UpperFormatter struct{}
func (UpperFormatter) Format(s string) string { return strings.ToUpper(s) }

// DO: Just use a function type or concrete type
type Formatter func(string) string
var UpperFormatter Formatter = strings.ToUpper
```

**Anti-pattern #2: Package Name Stuttering**

```go
// DON'T: "package server" with type "Server"
package server
type Server struct {}  // Called as server.Server

// DON'T: "package http" with type "HTTPClient" (redundant)
package http
type HTTPClient struct {}  // Called as http.HTTPClient

// DO: package server with type "Engine" or omit package prefix meaning
package server
type Engine struct {}  // Called as server.Engine
```

**Anti-pattern #3: Channel-Only Synchronization**

```go
// DON'T: Channel as a mutex
var done = make(chan struct{}, 1)
func increment() {
    done <- struct{}{}
    counter++
    <-done
}
// DO: Use sync.Mutex for mutual exclusion
var mu sync.Mutex
func increment() {
    mu.Lock()
    defer mu.Unlock()
    counter++
}
```

The channel-as-mutex anti-pattern is slower (channel operations involve scheduler handoffs), more complex, and doesn't support `TryLock` or RWMutex semantics.

**Anti-pattern #4: Global State Masquerading as Singleton**

```go
// DON'T: Global variable with init()
var db *sql.DB

func init() {
    var err error
    db, err = sql.Open("postgres", os.Getenv("DB_URL"))
    if err != nil {
        panic(err)
    }
}

// DO: Dependency injection through constructors
type Repository struct {
    db *sql.DB
}

func NewRepository(ctx context.Context, connStr string) (*Repository, error) {
    db, err := sql.Open("postgres", connStr)
    if err != nil {
        return nil, err
    }
    return &Repository{db: db}, nil
}
```

**Anti-pattern #5: Context Stored in Struct**

```go
// DON'T: Storing context
type Worker struct {
    ctx context.Context  // NEVER do this
}

// DO: Pass context as first parameter
func (w *Worker) Process(ctx context.Context, data []byte) error
```

Contexts are meant to be scoped to request lifetime, not stored. Storing them creates confusion about cancellation scope and can lead to goroutines using stale contexts.

### 6. Performance Considerations

**Cost of interface boxing:** Converting a concrete type to an interface requires allocating a new *iface* struct (two words: type pointer, data pointer). For value types, the data is copied to the heap if the interface escapes. Benchmark:

```go
func BenchmarkConcrete(b *testing.B) {
    var u User
    for i := 0; i < b.N; i++ {
        u.GetName()  // direct call, inlineable
    }
}

func BenchmarkInterface(b *testing.B) {
    var u interface{ GetName() string } = User{}
    for i := 0; i < b.N; i++ {
        u.GetName()  // dynamic dispatch, not inlined
    }
}
// Concrete: 0.35 ns/op, 0 allocs
// Interface: 2.8 ns/op, 0 allocs (but prevents inlining)
```

**Cost of `defer` in hot loops:** Before Go 1.14, `defer` had ~50ns overhead. Now it's ~6ns, but still not free. More importantly, defer in a loop creates a closure allocation:

```go
for i := 0; i < 1000000; i++ {
    defer func() { /* cleanup */ }() // Allocates closure each iteration
}
```

**Cost of anonymous goroutines:** Each `go func() { ... }()` allocates the closure on the heap if it captures variables. For short-lived goroutines, this stresses the GC:

```go
for _, item := range items {
    go func(i Item) { process(i) }(item)  // Closure allocates per iteration
}
// Better: Use worker pool (Chapter 25)
```

**Cost of `any` (empty interface):** When you store a concrete type in `any`, you lose type information, forcing runtime type assertions. Each type assertion is O(1) but includes a pointer comparison and type ID check. More importantly, `any` prevents compiler optimizations like bounds check elimination on slices.

### 7. Best Practices (The Idiomatic Way)

**The Single-Implementation Interface Rule:** Do not define an interface until you have at least two concrete implementations, or an explicit reason to mock (e.g., external API). This is "Interface segregation" taken practically:

```go
// Acceptable: Need to mock for testing
type UserGetter interface {
    GetUser(ctx context.Context, id string) (*User, error)
}
// Implementation: production (DB), test (mock)
```

**Accept interfaces, return structs (Airside's Rule):** This prevents leaking abstractions and keeps heap allocation predictable.

**Keep interfaces small:** The standard library's `io.Reader` (one method) and `io.Writer` (one method) are the gold standard.

**Name packages after what they provide, not what they contain:** `package http` provides HTTP clients and servers, not `package httpclient`. `package json` provides JSON encoding/decoding, not `package jsonutils`.

**Use `errgroup` for goroutine coordination instead of raw `sync.WaitGroup` + channels:**

```go
// Anti-pattern: Manual error propagation
var wg sync.WaitGroup
var errMu sync.Mutex
var firstErr error
for _, task := range tasks {
    wg.Add(1)
    go func(t Task) {
        defer wg.Done()
        if err := t.Run(); err != nil {
            errMu.Lock()
            if firstErr == nil {
                firstErr = err
            }
            errMu.Unlock()
        }
    }(task)
}
wg.Wait()

// Idiomatic:
g, ctx := errgroup.WithContext(ctx)
for _, task := range tasks {
    task := task
    g.Go(func() error {
        return task.RunWithContext(ctx)
    })
}
err := g.Wait()
```

**Prefer value receivers for small, immutable structs:** This avoids heap allocation and reduces GC pressure.

### 8. Examples

**Example 1: Refactoring over-abstraction**

```go
// Before: Java-style layering
type Repository interface {
    Find(id int) (interface{}, error)
}

type Service interface {
    GetData(id int) (interface{}, error)
}

type Controller struct {
    svc Service
}

// After: Direct, with only necessary abstraction
type UserGetter interface {
    GetUser(ctx context.Context, id int) (*User, error)
}

type Handler struct {
    getter UserGetter
}

func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    id, _ := strconv.Atoi(r.URL.Query().Get("id"))
    user, err := h.getter.GetUser(r.Context(), id)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    json.NewEncoder(w).Encode(user)
}
```

**Example 2: Fixing channel anti-pattern**

```go
// Before: Channel as error propagation mechanism
errCh := make(chan error, 1)
go func() {
    errCh <- doWork()
}()
select {
case err := <-errCh:
    // handle
case <-time.After(time.Second):
    // timeout
}

// After: Use context cancellation
ctx, cancel := context.WithTimeout(context.Background(), time.Second)
defer cancel()
err := doWorkWithContext(ctx)  // Function respects ctx.Done()
```

**Example 3: Avoiding init() magic**

```go
// Before: Silent initialization
var logger *slog.Logger
func init() {
    f, _ := os.Create("app.log")
    logger = slog.New(slog.NewJSONHandler(f, nil))
}

// After: Explicit initialization
func main() {
    logger, cleanup, err := setupLogging("app.log")
    if err != nil {
        log.Fatal(err)
    }
    defer cleanup()
    // use logger
}
```

### 9. Summary & Exercises

**Summary**

Anti-patterns in Go emerge when developers import paradigms from other languages (Java abstraction layers, JavaScript channel chaos, C++ deep generics) or misunderstand Go's runtime model. The key protections are:

1. **Resist premature interfaces** - Only abstract when you have multiple implementations or clear testing needs.
2. **Never store contexts** - Pass them explicitly as first parameters.
3. **Use the right concurrency primitive** - `sync.Mutex` for mutual exclusion, channels for communication/coordination, `errgroup` for bounded goroutine pools with error propagation.
4. **Avoid `init()` for complex setup** - Make initialization explicit and error-returning.
5. **Keep receivers consistent** - Either all value receivers or all pointer receivers for a given type.

Remember: Go's simplicity is a feature. If you find yourself writing a factory that builds a builder that configures a provider that instantiates a strategy—stop. Write a `New` function and a struct.

**Exercises**

**Exercise 1: Detect and refactor interface pollution**

Given the following codebase snapshot, identify which interfaces are justified (multiple implementations or testing need) and which are premature abstractions. Refactor the latter to concrete types:

```go
type Calculator interface {
    Add(a, b int) int
    Subtract(a, b int) int
}

type BasicCalc struct{}
func (BasicCalc) Add(a, b int) int { return a + b }
func (BasicCalc) Subtract(a, b int) int { return a - b }

type Stringifier interface {
    String() string
}

type User struct { Name string }
func (u User) String() string { return u.Name }

type Transformer interface {
    Transform(data []byte) ([]byte, error)
}

type UppercaseTransformer struct{}
func (UppercaseTransformer) Transform(data []byte) ([]byte, error) {
    return bytes.ToUpper(data), nil
}
```

**Exercise 2: Fix the goroutine leak and context misuse**

The following production code has a subtle leak when `request` processing times out. Fix it without changing the external API:

```go
type Processor struct {
    workerPool chan struct{} // buffered channel as semaphore
}

func (p *Processor) Process(ctx context.Context, req Request) (Response, error) {
    p.workerPool <- struct{}{} // acquire slot
    defer func() { <-p.workerPool }()
    
    resultCh := make(chan Response)
    go func() {
        resultCh <- p.doHeavyWork(req) // This blocks forever if parent context cancels
    }()
    
    select {
    case res := <-resultCh:
        return res, nil
    case <-ctx.Done():
        return Response{}, ctx.Err()
    }
}
```

**Exercise 3: Refactor "Java-style Go" to idiomatic Go**

The following code simulates a simple key-value store with unnecessary abstraction layers. Reduce it to the minimal necessary code while preserving functionality:

```go
type StorageEngine interface {
    Put(key, value string) error
    Get(key string) (string, error)
}

type InMemoryEngine struct {
    mu sync.RWMutex
    data map[string]string
}

func (e *InMemoryEngine) Put(key, value string) error {
    e.mu.Lock()
    defer e.mu.Unlock()
    e.data[key] = value
    return nil
}

func (e *InMemoryEngine) Get(key string) (string, error) {
    e.mu.RLock()
    defer e.mu.RUnlock()
    val, ok := e.data[key]
    if !ok {
        return "", errors.New("not found")
    }
    return val, nil
}

type CacheService interface {
    Set(key, value string) error
    Get(key string) (string, error)
}

type CacheServiceImpl struct {
    engine StorageEngine
}

func (c *CacheServiceImpl) Set(key, value string) error {
    return c.engine.Put(key, value)
}

func (c *CacheServiceImpl) Get(key string) (string, error) {
    return c.engine.Get(key)
}

type CacheFactory interface {
    Create() CacheService
}

type InMemoryFactory struct{}

func (f InMemoryFactory) Create() CacheService {
    return &CacheServiceImpl{
        engine: &InMemoryEngine{data: make(map[string]string)},
    }
}
```

*Hint: The entire system could be a single exported struct type.*
