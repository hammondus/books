## Chapter 32: Reflection & Unsafe

Go prides itself on being a simple, safe, compiled language. Yet the standard library ships with two packages that deliberately pierce that veil: `reflect` and `unsafe`. One allows you to examine and manipulate types and values at runtime; the other lets you subvert the type system entirely. This chapter treats them not as hammers in search of nails, but as escape hatches to be used sparingly—and often replaced by code generation.

---

### 1. Basic Usage

#### 1.1 The `reflect` Package

The two central concepts are `reflect.Type` (the Go representation of a type) and `reflect.Value` (a holder for a value of any type). You obtain them from any interface value.

```go
package main

import (
	"fmt"
	"reflect"
)

type Person struct {
	Name string `json:"name"`
	Age  int    `json:"age"`
}

func main() {
	p := Person{"Alice", 30}
	t := reflect.TypeOf(p)   // main.Person
	v := reflect.ValueOf(p)  // {Alice 30}

	fmt.Println("Type:", t)            // main.Person
	fmt.Println("Kind:", t.Kind())     // struct
	fmt.Println("NumField:", t.NumField())

	// Iterate fields
	for i := 0; i < t.NumField(); i++ {
		field := t.Field(i)
		value := v.Field(i)
		tag := field.Tag.Get("json")
		fmt.Printf("Field %d: Name=%s, Type=%v, Value=%v, json tag=%q\n",
			i, field.Name, field.Type, value, tag)
	}
}
```

You can also set values, but only if you have an addressable `reflect.Value`:

```go
p2 := Person{}
v2 := reflect.ValueOf(&p2).Elem()   // Elem() dereferences the pointer
nameField := v2.FieldByName("Name")
if nameField.CanSet() {
	nameField.SetString("Bob")
}
fmt.Println(p2.Name) // Bob
```

Reflection allows you to call methods dynamically, create new instances of a type, and inspect channels, maps, and slices.

#### 1.2 The `unsafe` Package

`unsafe` exposes operations that bypass Go’s type safety. The most common—and least dangerous—is the conversion between numeric types and `unsafe.Pointer`, and between different pointer types.

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	var x int64 = 0x0102030405060708
	ptr := unsafe.Pointer(&x)
	// Interpret the first byte as a uint8
	firstByte := *(*uint8)(ptr)
	fmt.Printf("First byte: %#x\n", firstByte) // platform-dependent (e.g., 0x08 on little-endian)

	// Access fields in a struct without importing the package that defines it
	type internal struct {
		a int
		b string
	}
	s := internal{42, "hello"}
	pb := (*string)(unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + unsafe.Offsetof(s.b)))
	fmt.Println(*pb) // "hello"
}
```

The rules for `unsafe.Pointer` are strict: a `uintptr` derived from a pointer does **not** keep the object alive; you must only use it immediately before converting back to an `unsafe.Pointer` while the original reference is still reachable.

---

### 2. Under the Hood

#### 2.1 Reflection’s Internal Mechanism

At runtime, every Go type has a descriptor—a `runtime._type` structure that stores size, kind, alignment, and method table. When you assign a value to an `interface{}`, the compiler builds a two‑word `eface` (or `iface` for non-empty interfaces): a pointer to the type descriptor and a pointer to the data. The `reflect` package simply exposes that internal representation. `reflect.TypeOf` returns the type descriptor, and `reflect.ValueOf` wraps the data pointer plus the type descriptor, while also tracking flags like “addressable” and “can set”.

Dynamic method dispatch via `reflect.Value.Call` works by constructing a call frame and invoking the function’s code pointer from the method table. Because `reflect` operates on `interface{}` values, every argument and return value must be boxed into interface headers, which causes allocations and indirection.

#### 2.2 The `unsafe` Package and the Compiler

The compiler treats `unsafe.Pointer` as a “universal reference” that can be converted to any pointer type. However, the `unsafe` package does **not** generate any additional runtime code; it’s a compile‑time gate. The real danger is that it disables the usual escape‑analysis and garbage‑collection safeguards.

The Go memory model guarantees that an object is alive as long as a pointer to it (including via `unsafe.Pointer`) is reachable. But a `uintptr` is a numeric integer; it doesn’t count as a reference. The classic mistake is storing a pointer as `uintptr`, doing some work that might trigger GC, and then converting back—the object may have moved or been collected. The `unsafe.Pointer` rules (documented in the package) define the only safe sequences: for example, `reflect.SliceHeader` or `reflect.StringHeader` manipulation must be done in a single expression that doesn’t let the original object become unreachable.

The runtime’s write barriers and stack maps completely ignore `uintptr`, so you must ensure that all accesses through `unsafe` are guarded by a live reference.

---

### 3. Why This Design?

Go exists to make large‑scale software engineering safe and productive. Reflection and `unsafe` are explicit contradictions of that mission—so why include them?

- **Reflection** is necessary for any library that must handle arbitrary types: JSON marshalers, SQL mappers, RPC frameworks, and IDE tooling. Without it, the standard library couldn’t offer `encoding/json` or `fmt.Sprintf("%+v")`. Go rejects macros and code generation as mandatory features of the language. Instead, it provides a runtime reflection mechanism that is opt‑in, explicit, and relatively slow—making it a deliberate trade‑off, not a default.

- **Unsafe** exists because Go lives in a real world of operating systems, C interop, and performance‑critical paths. The standard library itself uses `unsafe` to implement `sync.Pool`, `strings.Builder`, and `syscall`. The philosophy is: *Make the dangerous thing possible, but label it so clearly that no one can do it by accident.* You must literally type `import "unsafe"` and pass a `go vet` check that acknowledges its presence.

The Go team’s position is “reflection is never clear; use interfaces and code generation first.” The `unsafe` package is a necessary evil, not a feature.

---

### 4. Competing Approaches

#### Java / C\#

Both languages embed deep reflection support in the VM, including the ability to load and redefine classes at runtime. This enables powerful frameworks (Spring, Hibernate) but at a cost: every object carries a heavy class descriptor, and security managers must guard against reflective access. Go’s reflection is minimal—no method injection, no runtime code generation—and that reflects the “less is more” philosophy.

#### Python / Ruby

Dynamic languages have introspection as a first‑class citizen; you can add methods, change classes, and inspect frames on the fly. Go explicitly rejects that. Reflection is a separate, clunky package; you must consciously opt in, and the compiler can never be surprised by a type change.

#### C++

C++ offers compile‑time metaprogramming (templates, `constexpr`) and limited runtime type information (RTTI) via `typeid` and `dynamic_cast`. It has no standard reflection akin to `reflect.Value`. This means serialization and ORM-like libraries must either rely on code generators or on macro‑heavy frameworks. Go strikes a middle ground: code generation is the preferred path, but reflection is available for libraries that need it.

#### Rust

Rust avoids runtime reflection entirely; it relies on trait objects, procedural macros (`serde`), and compile‑time code generation. This yields zero‑cost abstractions and high safety. Go’s philosophy is similar—`go:generate` is the idiomatic answer to most problems that reflection would solve in Java or Python. `unsafe` in Rust is a scoped keyword; `unsafe` in Go is a whole package, but both signal “trust me” to the programmer.

---

### 5. Common Mistakes

1. **Using Reflection Where Interfaces Suffice**
   Developers from Java often reach for reflection to implement dynamic dispatch. In Go, a simple interface is type‑safe, fast, and readable. If you know the set of possible types, use a type switch or a generic function.

2. **Forgetting `CanSet` and Panicking**
   A `reflect.Value` obtained from a non‑addressable value (e.g., a map index, a return value) will have `CanSet() == false`. Calling `Set*` will panic. Always check:

   ```go
   if val.CanSet() {
       val.SetInt(42)
   }
   ```

3. **The `uintptr` Time Bomb**
   Storing a `uintptr` and then creating a new object or calling a function that might allocate can cause the GC to move the original object. The correct pattern is to keep the original pointer alive while the `uintptr` is in use, or to perform the conversion in a single, uninterrupted expression.

4. **Aliasing Structs with `unsafe` Without Alignment**
   Reinterpreting a `[]byte` as a struct can lead to unaligned accesses on some architectures (e.g., ARM), causing a crash or silent corruption. Always check `unsafe.Alignof`.

5. **Leaking Internal Details**
   Using `reflect` to expose unexported fields in a library breaks encapsulation and makes your API fragile. Use struct tags and exported interfaces instead.

6. **Assuming `reflect.DeepEqual` Is Fast or Always Correct**
   It’s recursive and uses reflection; it does not handle unexported fields and can be extremely slow. Prefer hand‑rolled comparison or `cmp` package from `golang.org/x` when performance matters.

---

### 6. Performance Considerations

Reflection is fundamentally expensive:

- Every `reflect.Value` is a small struct, but boxing a value into an interface to call `reflect.ValueOf` causes an allocation if the value doesn’t fit into a word.
- Method calls via `Call` involve creating a `[]reflect.Value` slice, unpacking arguments, and making an interface call. Expect a 10–100× slowdown compared to a direct call.
- Iterating over struct fields with `NumField` and `Field` is linear in the number of fields, but each access must fetch type information and check bounds.

`unsafe` operations, by contrast, are zero‑cost at runtime—they compile to a single pointer manipulation. However, they can **disable compiler optimizations**. For example, converting a `[]byte` to a `string` using `unsafe` (the classic `*(*string)(unsafe.Pointer(&b))` pattern) produces a string header that shares memory with the slice. This avoids an allocation and copy, but it may keep the entire backing array alive longer than necessary, preventing garbage collection of other parts of the slice’s buffer. The standard library’s `strings.Builder` now uses a similar trick internally, guarded by careful lifetime management.

> **Measurement is mandatory.** Use `go test -bench` and `pprof` to assess whether reflection is a bottleneck in your hot path. Often the answer is to generate code once at build time rather than reflect at runtime.

---

### 7. Best Practices

1. **Prefer Code Generation Over Reflection**
   The `go:generate` directive lets you run a Go program at build time that can produce type‑safe, high‑performance code. For example, a stringer, a JSON marshaler, or a deep‑copy function can all be generated from a simple struct definition. The `stringer` tool and `protobuf` compiler are canonical examples.

2. **Isolate Reflection in Small, Well‑Tested Functions**
   If you must use `reflect`, confine it to a single package boundary. Make the public API type‑safe and hide the `reflect` mess inside. The `encoding/json` package does exactly this: users never touch `reflect`.

3. **Always Check `IsValid` and `CanSet`**
   A zero `reflect.Value` is not valid; operations on it will panic. Guard any reflective operation with these checks.

4. **Use `unsafe` Only in Performance‑Critical, Well‑Understood Bottlenecks**
   And only if benchmarks prove that the safe, idiomatic version is unacceptable. When you do use `unsafe`, wrap it in a type‑safe wrapper that enforces invariants. For example, a function that returns a `[]byte` view of a string’s memory must document that the caller must not modify the slice.

5. **Leverage `unsafe.Pointer` Rules Religiously**
   Study the four allowed patterns in the `unsafe` package documentation. Never convert `uintptr` to `unsafe.Pointer` after the original pointer might have become unreachable.

6. **Use `reflect.Value.Interface()` to Escape Back**
   Once you’ve finished setting a value via reflection, you can call `.Interface()` to retrieve it as `any` and then type‑assert it back to the concrete type. This is often the safest way to return a reflected value to normal Go code.

7. **Don’t Expose `reflect` or `unsafe` in Your Public API**
   If a function signature includes `reflect.Type` or `unsafe.Pointer`, you’ve leaked an implementation detail that forces your callers to import these dangerous packages.

---

### 8. Examples

#### 8.1 A Simple Field‑Setter Using Reflection (with Safety)

```go
// SetField sets a struct field by name.
// The target must be a non-nil pointer to a struct.
// Returns an error if the field doesn't exist or cannot be set.
func SetField(target any, fieldName string, value any) error {
	v := reflect.ValueOf(target)
	if v.Kind() != reflect.Ptr || v.IsNil() {
		return fmt.Errorf("target must be a non-nil pointer")
	}
	v = v.Elem()
	if v.Kind() != reflect.Struct {
		return fmt.Errorf("target must point to a struct")
	}

	field := v.FieldByName(fieldName)
	if !field.IsValid() {
		return fmt.Errorf("no such field: %s", fieldName)
	}
	if !field.CanSet() {
		return fmt.Errorf("field %s cannot be set (unexported?)", fieldName)
	}

	newVal := reflect.ValueOf(value)
	if field.Type() != newVal.Type() {
		return fmt.Errorf("type mismatch: field %s is %s, got %s",
			fieldName, field.Type(), newVal.Type())
	}
	field.Set(newVal)
	return nil
}
```

#### 8.2 `go:generate` to Avoid Reflection

Suppose we have an enum and we want a `String()` method:

```go
// day.go
package main

//go:generate stringer -type=Day
type Day int

const (
	Monday Day = iota
	Tuesday
	Wednesday
)
```

Running `go generate` produces `day_string.go` with a fast, allocation‑free `func (Day) String() string` implemented with a static array, no reflection in sight.

#### 8.3 Unsafe: Zero‑copy Byte Slice to String

This pattern appears in high‑throughput I/O to avoid an allocation. **Only use it if the slice will not be modified afterward.**

```go
import "unsafe"

// BytesToString converts a byte slice to a string without copying.
// The caller guarantees that the underlying bytes will not change.
func BytesToString(b []byte) string {
	return *(*string)(unsafe.Pointer(&b))
}
```

This works because `[]byte` and `string` have identical header structures (a pointer and a length). It is used internally by the standard library in `strings.Builder`’s `String()` method. *Warning:* The result string shares memory with the slice; modifying the slice’s backing array will mutate the string’s contents, which is undefined behavior.

#### 8.4 Code Generation vs. Reflection: A Practical Comparison

Consider a simple function to print all fields of a struct with their `json` tags.

- **Reflection‑based** (slow, but works for any struct):

```go
func PrintJSONFields(v any) {
	t := reflect.TypeOf(v)
	for i := 0; i < t.NumField(); i++ {
		f := t.Field(i)
		fmt.Printf("%s -> %s\n", f.Name, f.Tag.Get("json"))
	}
}
```

- **Generated code** (fast, type‑safe, but requires a generator):

```go
// generated_struct_ printer.go
func (p *Person) PrintJSONFields() {
	fmt.Println("Name -> name")
	fmt.Println("Age -> age")
}
```

When building a production service that processes millions of records, generating the code once is a clear win.

---

### 9. Summary & Exercises

Go arms you with reflection for unavoidable runtime generality, `unsafe` for performance and interop, and `go:generate` to replace both whenever possible. The “Go way” is to resist the temptation of clever dynamic code and instead use compile‑time code generation. When you do reach for `reflect`, isolate it; when you reach for `unsafe`, wrap it.

**Key Takeaways:**

- Reflection is the machinery behind serialization, but it is slow, panicky, and verbose.
- `unsafe` bypasses the type system but demands a deep understanding of memory layout and GC.
- Code generation (`go:generate`) gives you the speed and safety of handwritten code with the automation of reflection.

**Exercises:**

1. **Type‑Safe Cache with Fallback to Reflection**
   Build a concurrent cache using generics (`Cache[K, V]`) that stores any value. Then write a `SnapShot()` method that uses reflection (or a type switch) to produce a pretty‑printed string representation of the cache contents. Benchmark the reflective path against a version that requires `V` to implement `fmt.Stringer`. Document when you would drop the reflective fallback.

2. **Unsafe Struct Hashing**
   Write a function `HashStruct(v any) uint64` that uses `unsafe` to treat a struct’s memory as a `[]byte` and passes it to a hash function (like `hash/fnv`). Ensure you handle alignment and that the GC doesn’t move the struct during hashing. Compare its performance with a safe, field‑by‑field hash using a code‑generated approach.

3. **Build a Code Generator**
   Design a simple command‑line tool that reads a Go source file and generates a `Validate() error` method for any struct that has a `validate` tag (e.g., `validate:"required,min=1"`). The generated code should run without importing `reflect`. Integrate it via `go:generate` in a sample project.
