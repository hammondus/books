# Chapter 32: Reflection & Unsafe

## Basic Usage

Reflection in Go, provided by the `reflect` package, allows you to inspect and dynamically manipulate values, types, and structures at runtime. The `unsafe` package, true to its name, bypasses Go's type safety and memory safety guarantees for extreme performance or interoperability scenarios.

### Basic Reflection Operations

```go
package main

import (
    "fmt"
    "reflect"
)

type User struct {
    ID    int    `json:"id" db:"user_id"`
    Name  string `json:"name" db:"user_name"`
    Email string `json:"email" db:"email_addr"`
}

func inspectValue(v any) {
    // Get reflection information
    val := reflect.ValueOf(v)
    typ := reflect.TypeOf(v)
    
    fmt.Printf("Type: %v\n", typ)
    fmt.Printf("Kind: %v\n", typ.Kind())
    fmt.Printf("Value: %v\n", val)
    
    // Check if it's a pointer
    if typ.Kind() == reflect.Ptr {
        fmt.Printf("Points to: %v\n", typ.Elem())
        fmt.Printf("Is nil pointer? %v\n", val.IsNil())
    }
}

func modifyField(ptr any, fieldName string, newValue any) error {
    val := reflect.ValueOf(ptr)
    
    // Must be pointer to struct
    if val.Kind() != reflect.Ptr || val.Elem().Kind() != reflect.Struct {
        return fmt.Errorf("expected pointer to struct, got %T", ptr)
    }
    
    structVal := val.Elem()
    field := structVal.FieldByName(fieldName)
    
    if !field.IsValid() {
        return fmt.Errorf("field %s not found", fieldName)
    }
    
    if !field.CanSet() {
        return fmt.Errorf("field %s cannot be set (unexported?)", fieldName)
    }
    
    newVal := reflect.ValueOf(newValue)
    if field.Type() != newVal.Type() {
        return fmt.Errorf("type mismatch: expected %v, got %v", 
            field.Type(), newVal.Type())
    }
    
    field.Set(newVal)
    return nil
}

func main() {
    user := User{ID: 1, Name: "Alice", Email: "alice@example.com"}
    
    // Inspect struct tags
    t := reflect.TypeOf(user)
    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        fmt.Printf("Field: %s, JSON tag: %s, DB tag: %s\n",
            field.Name, field.Tag.Get("json"), field.Tag.Get("db"))
    }
    
    // Modify field dynamically
    fmt.Printf("Before: %+v\n", user)
    if err := modifyField(&user, "Name", "Bob"); err != nil {
        panic(err)
    }
    fmt.Printf("After: %+v\n", user)
}
```

### Basic Unsafe Operations

```go
package main

import (
    "fmt"
    "unsafe"
)

func main() {
    // Converting between types (dangerous)
    var f float64 = 3.14159
    
    // Treat float64 as uint64 (bit pattern)
    bits := *(*uint64)(unsafe.Pointer(&f))
    fmt.Printf("Float %v as bits: %#x\n", f, bits)
    
    // Working with struct offsets
    type Config struct {
        Debug bool
        Port  int
        Host  [64]byte
    }
    
    cfg := Config{Debug: true, Port: 8080}
    
    // Manual pointer arithmetic (rarely needed, but possible)
    portPtr := (*int)(unsafe.Pointer(
        uintptr(unsafe.Pointer(&cfg)) + unsafe.Offsetof(cfg.Port),
    ))
    fmt.Printf("Original port: %d\n", *portPtr)
    
    *portPtr = 9090
    fmt.Printf("Modified config: %+v\n", cfg)
    
    // Slice header manipulation (use with extreme caution)
    data := []byte("hello")
    
    // Get slice header
    header := (*reflect.SliceHeader)(unsafe.Pointer(&data))
    fmt.Printf("Slice: ptr=%#x, len=%d, cap=%d\n", 
        header.Data, header.Len, header.Cap)
}
```

## Under the Hood

### Reflection Internals

Reflection works because the Go runtime stores type metadata alongside every value. When you call `reflect.TypeOf()`, you're accessing this metadata through a `reflect.rtype` structure.

```go
// Simplified representation of what the runtime stores
type rtype struct {
    size       uintptr
    ptrdata    uintptr
    hash       uint32
    tflag      uint8
    align      uint8
    fieldAlign uint8
    kind       uint8
    // ... more fields for methods, fields, etc.
}
```

The `reflect.Value` struct holds both a pointer to the actual data and a pointer to its type descriptor:

```go
type Value struct {
    typ *rtype      // Type information
    ptr unsafe.Pointer // Actual data
    flag uintptr     // Metadata (addressable, exported, etc.)
}
```

When you call `ValueOf()`, Go:
1. Checks that the value is addressable (has memory location)
2. Extracts the type information from the interface's itab/metadata
3. Creates a `reflect.Value` that can be used for inspection/modification

**Key insight:** Reflection forces escaped allocations. Any value passed to `reflect.ValueOf` escapes to the heap because the compiler cannot guarantee the reflection won't outlive the stack frame.

### Unsafe Package Internals

The `unsafe` package is tiny (only 9 functions/types in Go 1.21+), but its implications are massive. It essentially disables the compiler's safety checks:

```go
// Package unsafe contains operations that step around the type safety of Go.
package unsafe

type ArbitraryType int
type Pointer *ArbitraryType

func Sizeof(x ArbitraryType) uintptr
func Offsetof(x ArbitraryType) uintptr
func Alignof(x ArbitraryType) uintptr
```

**The critical rule:** `unsafe.Pointer` is a pointer that can hold any type, but the GC still treats it as a pointer. The actual danger comes from converting to `uintptr`, which is an integer that the GC doesn't trace.

```go
// Safe: GC tracks p
p := unsafe.Pointer(&x)

// DANGEROUS: GC does NOT track uintptr
u := uintptr(unsafe.Pointer(&x))
// If GC runs now, x could be moved (if compacting GC existed) or collected
```

**Memory layout guarantee:** Go's compiler ensures struct fields are laid out in declaration order, but padding may be inserted for alignment. `unsafe.Offsetof` gives you the actual byte offset after padding.

## Why This Design?

### Reflection Philosophy: "Necessary Evil with Constraints"

The Go team added reflection reluctantly, recognizing two unavoidable needs:
1. **Serialization** (JSON, XML, database drivers) cannot know types at compile time
2. **Tooling** (go test, go fmt, debugging) needs to inspect arbitrary values

But they deliberately constrained reflection to prevent it from becoming a crutch:

**Why no `settable` by default?** Reflection can only modify values that are "addressable" (have a memory location). This prevents:
- Modifying temporary values (`reflect.ValueOf(42).Set(…)` - illegal)
- Modifying unexported struct fields without `CanSet` check
- Creating values from nothing (enforces type safety)

**Why slow by design?** The Go team explicitly prioritized compile-time type safety. Reflection is intentionally verbose and somewhat slow to discourage overuse. If reflection were fast and ergonomic, developers would use it instead of proper static typing.

### Unsafe Philosophy: "Power, Not Safety"

The `unsafe` package exists for three specific scenarios the Go team couldn't ignore:

1. **System programming** (interfacing with OS, C libraries, hardware)
2. **Extreme optimization** (avoiding allocations in hot paths)
3. **Data structure internals** (implementing sync.Map, generic containers before generics)

**The naming is intentional:** `unsafe` isn't hidden or underdocumented. The stark name is a warning. Every use requires a comment explaining why it's necessary.

**Why not remove it?** Because the legitimate use cases are real. The Go runtime itself uses `unsafe` extensively internally. The compromise: make it available but force users to opt-in explicitly and understand the risks.

## Competing Approaches

### Reflection Comparison

| Language | Approach | Trade-offs |
|----------|----------|------------|
| **Go** | `reflect` package, explicit opt-in | Verbose but explicit. Runtime type info always available (memory cost). |
| **Java** | Runtime reflection via `Class<?>` | Extremely powerful but slow. Can access private fields (security issues). |
| **Rust** | Limited reflection via `std::any::TypeId` + traits | No full reflection by design. Relies on macros (compile-time code generation). |
| **Python** | Everything is runtime reflection | Flexible but slow. No compile-time safety. |
| **C++** | RTTI (minimal) + manual type erasure | You pay for what you use. No comprehensive reflection until C++26/reflection TS. |

**Go's unique stance:** Unlike Java, Go's reflection cannot access unexported fields without hacks. Unlike Rust, Go's reflection doesn't require explicit opt-in for every type. Go chose middle ground: reflection works on all types, but limited to public API.

### Unsafe Comparison

| Language | Approach | Safety Mechanism |
|----------|----------|------------------|
| **Go** | `unsafe` package (opt-in footgun) | No safety, but naming and docs warn heavily |
| **Rust** | `unsafe` keyword blocks | Compiler enforces unsafe blocks contain the danger, everything else safe |
| **C/C++** | Everything is unsafe by default | No safety unless you manually add checks |
| **Zig** | `@intToPtr` etc., explicit casting | Unsafe is explicit but ergonomic |

**Critical difference from Rust:** Rust's `unsafe` still prevents many UB scenarios (data races, use-after-free). Go's `unsafe` is truly unsafe—you can break everything.

## Common Mistakes

### Reflection Mistakes

**1. Forgetting `CanSet` check**
```go
// WRONG: Panics because field is unexported
type Person struct { name string }
p := Person{name: "Alice"}
v := reflect.ValueOf(&p).Elem().FieldByName("name")
v.SetString("Bob") // PANIC: cannot set unexported field

// RIGHT: Check first
if v.CanSet() {
    v.SetString("Bob")
} else {
    // Use other approach or return error
}
```

**2. Assuming `Elem()` works on non-pointers**
```go
// WRONG: Panics if v is not a pointer or interface
func setField(v reflect.Value, val any) {
    v.Elem().Set(reflect.ValueOf(val)) // PANIC if v is struct
}

// RIGHT: Check kind first
func setField(v reflect.Value, val any) error {
    if v.Kind() != reflect.Ptr {
        return fmt.Errorf("need pointer, got %v", v.Kind())
    }
    if v.Elem().Kind() != reflect.Struct {
        return fmt.Errorf("need struct, got %v", v.Elem().Kind())
    }
    v.Elem().Set(reflect.ValueOf(val))
    return nil
}
```

**3. Performance death by reflection in hot paths**
```go
// WRONG: Using reflection to call a simple getter 10M times
for i := 0; i < 10_000_000; i++ {
    val := reflect.ValueOf(obj).MethodByName("GetID").Call(nil)
    ids = append(ids, val[0].Int())
}
// This is ~100x slower than direct call

// RIGHT: Cache the method
getter := reflect.ValueOf(obj).MethodByName("GetID")
for i := 0; i < 10_000_000; i++ {
    val := getter.Call(nil)
    ids = append(ids, val[0].Int())
}
// Still ~50x slower than direct call - reconsider approach
```

### Unsafe Mistakes

**1. Storing `uintptr` across GC cycles**
```go
// DISASTER: GC can move memory between these lines
ptr := unsafe.Pointer(&x)
uptr := uintptr(ptr)
// GC could run here, moving x to new address
newPtr := unsafe.Pointer(uptr) // NOW POINTS TO WRONG MEMORY
```

**2. Incorrect alignment calculation**
```go
// WRONG: Assumes no padding
type Bad struct {
    b byte   // offset 0
    i int64  // offset 1 (WRONG - alignment requires offset 8)
}
// Using unsafe.Offsetof manually without considering compiler padding

// RIGHT: Let compiler tell you
offset := unsafe.Offsetof(Bad{}.i) // Returns 8, not 1
```

**3. Breaking escape analysis**
```go
// WRONG: Prevents legitimate escape analysis optimizations
func badAlloc() *int {
    x := 42
    return (*int)(unsafe.Pointer(&x)) // Forces x to heap but...
    // Compiler can't track this pointer correctly
}

// RIGHT: Use normal returns for heap allocation
func goodAlloc() *int {
    x := new(int)
    *x = 42
    return x
}
```

**4. Using `unsafe.Pointer` for CGO without proper pinning**
```go
// DANGEROUS: Go garbage collector can move memory
data := make([]byte, 1024)
C.process(unsafe.Pointer(&data[0])) // If GC runs, data might move

// SAFER: Pin memory (Go 1.21+)
runtime.Pin(&data[0])
defer runtime.Unpin(&data[0])
C.process(unsafe.Pointer(&data[0]))
```

## Performance Considerations

### Reflection Performance Characteristics

| Operation | Approximate Cost (relative to direct) | Allocation Behavior |
|-----------|--------------------------------------|---------------------|
| `TypeOf` | ~50x | No allocations |
| `ValueOf` (non-pointer) | ~100x | Escapes to heap |
| `FieldByName` | ~200x | Linear scan of fields |
| `MethodByName` | ~1000x | Linear scan of methods |
| `Call` (method) | ~1000x + per-call overhead | Argument boxing allocations |
| `Set` on primitive | ~30x | No allocations |
| `Set` on struct | ~500x | Requires new value allocation |

**The real cost:** Reflection forces deoptimization. The compiler's inliner, escape analyzer, and optimization passes all give up when reflection is involved.

### Memory Impact of Reflection

```go
// Each reflect.Value holds ~24-48 bytes of metadata
var cache map[string]reflect.Value // BAD: 1000 entries = 48KB overhead

// Better: Cache only what you need
type CachedField struct {
    Index int
    Type  reflect.Type
}
var fieldCache map[string]CachedField // ~16 bytes per entry
```

### Unsafe Performance Gains (When Used Correctly)

**Zero-copy byte to string conversion:**
```go
// Standard way: allocates
func bytesToString(b []byte) string {
    return string(b) // Allocates new string
}

// Unsafe way: zero-copy (but string becomes mutable via original slice!)
func bytesToStringUnsafe(b []byte) string {
    return *(*string)(unsafe.Pointer(&b)) // No allocation
}
// Benchmark: 0.3 ns/op vs 15 ns/op for large slices
```

**Avoiding interface boxing in tight loops:**
```go
// With interface boxing (allocates)
var vals []any = make([]any, 1000)
for i := range vals {
    vals[i] = i // Each int escapes to heap
}

// With unsafe and typed array (no allocations)
var ints []int = make([]int, 1000)
ptr := unsafe.Pointer(&ints[0])
// Process as int array directly
```

**The reality:** Most `unsafe` optimizations save 10-50ns per operation. This only matters if you're doing millions of operations per second. For normal code, the complexity isn't worth it.

## Best Practices

### Reflection Best Practices

**1. Prefer code generation over reflection**
```go
// BAD: Reflection for repetitive tasks
func validateOrder(order Order) []error {
    var errs []error
    v := reflect.ValueOf(order)
    t := v.Type()
    for i := 0; i < v.NumField(); i++ {
        field := v.Field(i)
        tag := t.Field(i).Tag.Get("validate")
        if tag == "required" && field.IsZero() {
            errs = append(errs, fmt.Errorf("%s is required", t.Field(i).Name))
        }
    }
    return errs
}

// GOOD: Code generation (go generate)
//go:generate stringer -type=OrderStatus
//go:generate genvalidate -type=Order
```

**2. Cache reflection results at init time**
```go
var (
    userType   = reflect.TypeOf(User{})
    fieldCache = sync.Map{} // field name -> index
)

func init() {
    for i := 0; i < userType.NumField(); i++ {
        field := userType.Field(i)
        fieldCache.Store(field.Name, i)
    }
}

func setFieldFast(u *User, name string, val any) error {
    idxAny, ok := fieldCache.Load(name)
    if !ok {
        return fmt.Errorf("field not found: %s", name)
    }
    idx := idxAny.(int)
    reflect.ValueOf(u).Elem().Field(idx).Set(reflect.ValueOf(val))
    return nil
}
```

**3. Type switches instead of reflection when possible**
```go
// BAD: Reflection for type dispatch
func toString(v any) string {
    rv := reflect.ValueOf(v)
    switch rv.Kind() {
    case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
        return strconv.FormatInt(rv.Int(), 10)
    // ... many cases
    }
}

// GOOD: Type switch (faster, clearer)
func toString(v any) string {
    switch x := v.(type) {
    case int: return strconv.Itoa(x)
    case int64: return strconv.FormatInt(x, 10)
    case string: return x
    // ...
    }
}
```

### Unsafe Best Practices

**1. Isolate unsafe code behind safe APIs**
```go
// Package fastbytes provides zero-copy conversions
package fastbytes

// StringToBytes converts string to byte slice without allocation.
// WARNING: The returned byte slice MUST NOT be modified, as it shares
// memory with the original string.
func StringToBytes(s string) []byte {
    if s == "" {
        return []byte{}
    }
    return unsafe.Slice(unsafe.StringData(s), len(s))
}

// BytesToString converts byte slice to string without allocation.
// WARNING: The byte slice MUST NOT be modified after conversion.
func BytesToString(b []byte) string {
    if len(b) == 0 {
        return ""
    }
    return unsafe.String(&b[0], len(b))
}
```

**2. Document every unsafe.Operation with a safety justification**
```go
// MUST: Comment explaining WHY unsafe is necessary
func fastIPToUint32(ip net.IP) uint32 {
    // IPv4 addresses are guaranteed to be 4 bytes in net.IP representation
    // when parsing from IPv4 strings. This unsafe conversion avoids interface
    // boxing and bounds checking, saving ~15ns per call in packet processing.
    // This function is called 1M+ times per second, making this worthwhile.
    ipv4 := ip.To4()
    if ipv4 == nil {
        return 0
    }
    return *(*uint32)(unsafe.Pointer(&ipv4[0]))
}
```

**3. Use modern unsafe utilities (Go 1.20+)**
```go
// Go 1.17 and earlier: manual slice header manipulation
oldWay := *(*[]byte)(unsafe.Pointer(&header))

// Go 1.20+: safe slice utilities (less dangerous)
data := unsafe.Slice(ptr, length)      // Convert pointer to slice
str := unsafe.String(ptr, length)      // Convert pointer to string
ptr := unsafe.StringData(str)          // Get string's internal pointer

// These are still unsafe but less likely to cause memory corruption
```

**4. Never use unsafe for "convenience"**
```go
// WRONG: Used because you're lazy about type conversion
func addToMap(m map[string]any, key string, value any) {
    // Just use normal type assertion!
    m[key] = value
}

// RIGHT: Only when you have a performance profile proving it matters
var globalMap sync.Map // 10M operations per second
func addOptimized(key, value unsafe.Pointer) {
    // Only because we measured and proved this was the bottleneck
}
```

## Examples

### Example 1: Generic Struct Validator Using Reflection

```go
package validator

import (
    "fmt"
    "reflect"
    "regexp"
    "strings"
)

type ValidationRule string

const (
    Required ValidationRule = "required"
    Email    ValidationRule = "email"
    MinLen   ValidationRule = "minlen"
)

type ValidationError struct {
    Field string
    Rule  string
    Value any
    Message string
}

func (e ValidationError) Error() string {
    return e.Message
}

func ValidateStruct(s any) []error {
    var errors []error
    v := reflect.ValueOf(s)
    t := v.Type()
    
    if t.Kind() == reflect.Ptr {
        v = v.Elem()
        t = t.Elem()
    }
    
    if t.Kind() != reflect.Struct {
        return []error{fmt.Errorf("expected struct, got %s", t.Kind())}
    }
    
    for i := 0; i < v.NumField(); i++ {
        field := v.Field(i)
        fieldType := t.Field(i)
        
        // Parse validation tags
        tag := fieldType.Tag.Get("validate")
        if tag == "" || tag == "-" {
            continue
        }
        
        rules := strings.Split(tag, ",")
        for _, rule := range rules {
            if err := validateField(field, fieldType.Name, rule); err != nil {
                errors = append(errors, err)
            }
        }
    }
    
    return errors
}

func validateField(val reflect.Value, fieldName, rule string) error {
    // Handle pointer fields
    if val.Kind() == reflect.Ptr && val.IsNil() {
        if rule == "required" {
            return ValidationError{
                Field: fieldName,
                Rule: rule,
                Value: nil,
                Message: fmt.Sprintf("%s is required", fieldName),
            }
        }
        return nil
    }
    
    if val.Kind() == reflect.Ptr {
        val = val.Elem()
    }
    
    switch rule {
    case "required":
        if isZero(val) {
            return ValidationError{
                Field: fieldName,
                Rule: rule,
                Value: val.Interface(),
                Message: fmt.Sprintf("%s is required", fieldName),
            }
        }
        
    case "email":
        if val.Kind() != reflect.String {
            return nil
        }
        str := val.String()
        if str != "" {
            emailRegex := regexp.MustCompile(`^[a-z0-9._%+\-]+@[a-z0-9.\-]+\.[a-z]{2,}$`)
            if !emailRegex.MatchString(str) {
                return ValidationError{
                    Field: fieldName,
                    Rule: rule,
                    Value: str,
                    Message: fmt.Sprintf("%s must be a valid email address", fieldName),
                }
            }
        }
        
    default:
        // Handle minlen=N
        if strings.HasPrefix(rule, "minlen=") {
            if val.Kind() != reflect.String && val.Kind() != reflect.Slice && val.Kind() != reflect.Array {
                return nil
            }
            minLen := 0
            fmt.Sscanf(rule, "minlen=%d", &minLen)
            if val.Len() < minLen {
                return ValidationError{
                    Field: fieldName,
                    Rule: rule,
                    Value: val.Interface(),
                    Message: fmt.Sprintf("%s must have at least %d characters/items", fieldName, minLen),
                }
            }
        }
    }
    
    return nil
}

func isZero(val reflect.Value) bool {
    return val.Interface() == reflect.Zero(val.Type()).Interface()
}

// Usage
type User struct {
    Name  string `validate:"required,minlen=2"`
    Email string `validate:"email"`
    Age   int    `validate:"required"`
}

func main() {
    user := User{Name: "A", Email: "invalid", Age: 0}
    if errs := ValidateStruct(user); len(errs) > 0 {
        for _, err := range errs {
            fmt.Println(err)
        }
    }
    // Output:
    // Name must have at least 2 characters/items
    // Email must be a valid email address
    // Age is required
}
```

### Example 2: Debug Printer with Reflection

```go
package debug

import (
    "fmt"
    "reflect"
    "strings"
)

// PrettyPrint prints any Go value with indentation and type information
func PrettyPrint(v any, indent string) string {
    return prettyPrint(reflect.ValueOf(v), indent, 0)
}

func prettyPrint(val reflect.Value, indent string, depth int) string {
    currentIndent := strings.Repeat(indent, depth)
    nextIndent := strings.Repeat(indent, depth+1)
    
    switch val.Kind() {
    case reflect.Invalid:
        return "nil"
        
    case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
        return fmt.Sprintf("%d", val.Int())
        
    case reflect.String:
        return fmt.Sprintf("%q", val.String())
        
    case reflect.Bool:
        return fmt.Sprintf("%t", val.Bool())
        
    case reflect.Ptr:
        if val.IsNil() {
            return "nil"
        }
        return "&" + prettyPrint(val.Elem(), indent, depth)
        
    case reflect.Struct:
        var b strings.Builder
        b.WriteString(val.Type().Name() + "{\n")
        for i := 0; i < val.NumField(); i++ {
            field := val.Type().Field(i)
            if !field.IsExported() {
                continue
            }
            fieldVal := val.Field(i)
            b.WriteString(fmt.Sprintf("%s%s: %s,\n",
                nextIndent, field.Name, prettyPrint(fieldVal, indent, depth+1)))
        }
        b.WriteString(currentIndent + "}")
        return b.String()
        
    case reflect.Slice, reflect.Array:
        if val.Len() == 0 {
            return fmt.Sprintf("%s[]", val.Type().Elem().Kind().String())
        }
        var b strings.Builder
        b.WriteString("[\n")
        for i := 0; i < val.Len(); i++ {
            b.WriteString(fmt.Sprintf("%s%d: %s,\n",
                nextIndent, i, prettyPrint(val.Index(i), indent, depth+1)))
        }
        b.WriteString(currentIndent + "]")
        return b.String()
        
    case reflect.Map:
        if val.Len() == 0 {
            return "map[]"
        }
        var b strings.Builder
        b.WriteString("map[" + val.Type().Key().Kind().String() + "]" + val.Type().Elem().Kind().String() + "{\n")
        for _, key := range val.MapKeys() {
            b.WriteString(fmt.Sprintf("%s%s: %s,\n",
                nextIndent, prettyPrint(key, indent, depth+1), 
                prettyPrint(val.MapIndex(key), indent, depth+1)))
        }
        b.WriteString(currentIndent + "}")
        return b.String()
        
    default:
        return fmt.Sprintf("%v", val.Interface())
    }
}

// Usage
type Config struct {
    Name    string
    Port    int
    Enabled bool
    Tags    []string
    Metadata map[string]any
}

func main() {
    cfg := Config{
        Name:    "api-server",
        Port:    8080,
        Enabled: true,
        Tags:    []string{"production", "v2"},
        Metadata: map[string]any{
            "owner": "platform-team",
            "backup": true,
        },
    }
    
    fmt.Println(PrettyPrint(cfg, "  "))
    // Output:
    // Config{
    //   Name: "api-server",
    //   Port: 8080,
    //   Enabled: true,
    //   Tags: [
    //     0: "production",
    //     1: "v2",
    //   ],
    //   Metadata: map[string]interface {}{
    //     "backup": true,
    //     "owner": "platform-team",
    //   },
    // }
}
```

### Example 3: Zero-Copy Byte Buffer Pool with Unsafe

```go
package zcopy

import (
    "sync"
    "unsafe"
)

// ByteBufferPool provides zero-copy byte slice management
type ByteBufferPool struct {
    pool sync.Pool
    size int
}

func NewByteBufferPool(size int) *ByteBufferPool {
    return &ByteBufferPool{
        pool: sync.Pool{
            New: func() any {
                buf := make([]byte, size)
                return &buf
            },
        },
        size: size,
    }
}

// Get retrieves a byte slice of the specified size
func (p *ByteBufferPool) Get() []byte {
    bufPtr := p.pool.Get().(*[]byte)
    return *bufPtr
}

// Put returns the byte slice to the pool after resetting
func (p *ByteBufferPool) Put(buf []byte) {
    if cap(buf) < p.size {
        return // Don't pool undersized buffers
    }
    // Reset using unsafe to avoid allocation
    for i := range buf {
        buf[i] = 0
    }
    p.pool.Put(&buf)
}

// ZeroCopyStringToBytes converts string to byte slice without allocation
// WARNING: The returned slice shares memory with the string and MUST NOT be modified
func ZeroCopyStringToBytes(s string) []byte {
    if s == "" {
        return nil
    }
    return unsafe.Slice(unsafe.StringData(s), len(s))
}

// ZeroCopyBytesToString converts byte slice to string without allocation
// WARNING: The byte slice MUST NOT be modified after conversion
func ZeroCopyBytesToString(b []byte) string {
    if len(b) == 0 {
        return ""
    }
    return unsafe.String(&b[0], len(b))
}

// Benchmark comparison:
// BenchmarkStandardBytesToString-8    50000000    25.4 ns/op    16 B/op    1 allocs/op
// BenchmarkZeroCopyBytesToString-8    1000000000  0.31 ns/op    0 B/op     0 allocs/op

// Production example: High-throughput HTTP header parsing
type HeaderParser struct {
    pool *ByteBufferPool
}

func (p *HeaderParser) ParseHeader(data []byte) (key, value string) {
    // Find colon without allocations
    for i, b := range data {
        if b == ':' {
            // Zero-copy conversion
            key = ZeroCopyBytesToString(data[:i])
            value = ZeroCopyBytesToString(data[i+1:])
            return
        }
    }
    return "", ""
}
```

## Summary & Exercises

### Summary

Reflection and unsafe represent Go's escape hatches—powerful tools that violate normal language guarantees for specific, narrow use cases.

**Reflection's sweet spot:**
- Serialization frameworks (JSON, XML, Protocol Buffers)
- Generic algorithms operating on unknown types
- Testing and debugging tools
- ORMs and database mappers

**Reflection's danger zones:**
- Hot paths (performance degradation)
- APIs where static typing would work (complexity cost)
- Code that must be maintained by others (readability cost)

**Unsafe's legitimate uses:**
- System programming (CGO, OS interfaces, hardware access)
- Zero-copy optimizations (proven by profiling)
- Implementing high-performance data structures

**Unsafe's forbidden uses:**
- Avoiding proper error handling
- "Clever" code that's hard to understand
- Optimizations without benchmark evidence

**The Go mantra applied:** "A little copying is better than a little dependency" extends to "A little explicit code is better than a little reflection." Only reach for reflection when static code generation or interfaces cannot solve the problem. Only reach for unsafe when you can prove it matters.

### Exercises

#### Exercise 1: Generic Deep Equal Without Reflection (Advanced)

Implement a `DeepEqual` function that compares two arbitrary values for equality **without using `reflect.DeepEqual`**. You may use limited reflection for type inspection, but implement the actual comparison logic.

```go
func DeepEqual(a, b any) bool {
    // Handle nil, basic types, slices, maps, structs, and cycles
    // Must detect circular references to avoid infinite recursion
    // Performance: within 2x of reflect.DeepEqual on comparable structures
}

// Bonus: Add thread safety and handle unexported fields appropriately
```

**Requirements:**
- Detect cycles (use `sync.Map` or `unsafe.Pointer` tracking)
- Handle maps with non-comparable keys (by converting to strings)
- Respect `Equal` method if implemented (like `cmp.Equal` from google/cmp)

#### Exercise 2: Tag-Based Environment Variable Mapper (Practical)

Build a function that populates struct fields from environment variables using struct tags. This should handle nesting, pointers, and slices.

```go
type Config struct {
    Server struct {
        Host string `env:"SERVER_HOST" default:"localhost"`
        Port int    `env:"SERVER_PORT" default:"8080"`
    }
    Database struct {
        URL      string `env:"DB_URL" required:"true"`
        PoolSize int    `env:"DB_POOL_SIZE" default:"10"`
    }
    Features []string `env:"FEATURES" sep:","`
}

func LoadEnvConfig(prefix string, cfg any) error {
    // Load from os.Getenv() and populate cfg
    // Required fields must be set, or return error
    // Support default values
    // Support slices (parse comma-separated)
    // Support nested structs with prefix chaining
}

// Challenge: Support environment variable expansion like "host:${DB_HOST}"
```

**Acceptance criteria:**
- Works with any struct depth
- Proper error messages indicating which field/environment variable is missing
- Performance: can load 10,000 config structs per second
- Test coverage for all edge cases

#### Exercise 3: Type-Safe Unsafe Arena Allocator (Expert)

Implement a memory arena that allocates objects of any type in a contiguous byte slice, bypassing the GC for performance. The arena must be type-safe through a generic API.

```go
type Arena struct {
    // Implementation using []byte and unsafe
}

func NewArena(size int) *Arena

func (a *Arena) New[T any]() *T {
    // Allocate space for T in the arena, return pointer
    // Must handle alignment correctly (use unsafe.Alignof)
    // Must panic if arena is full
}

func (a *Arena) Reset() {
    // Clear all allocations without freeing memory
    // Next New() will start from beginning
}

// Challenge: Add slice allocation
func (a *Arena) MakeSlice[T any](len, cap int) []T
```

**Constraints:**
- No heap allocations after arena creation (except for the arena itself)
- Must be safe for concurrent use (add synchronization)
- Must handle structs with pointers (those pointers must also be in arena or nil)
- Provide benchmark showing 10x+ improvement over normal allocation for high-frequency object creation

**Safety requirements:**
- Use build tags to make this debug-mode safe (add bounds checks)
- Document all unsafe operations with justification
- Provide test that shows memory safety (no overlapping allocations, correct alignment)

**Extra credit:** Implement generational arena with version stamps to detect use-after-free bugs.
