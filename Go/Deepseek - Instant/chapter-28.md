## Chapter 28: JSON & Data Serialization

JSON has become the lingua franca of web APIs, configuration files, and data pipelines. Go’s `encoding/json` package is a testament to the language’s philosophy: it prioritizes simplicity and safety while offering escape hatches for performance-critical paths. This chapter dissects how Go serializes structured data, why the design choices were made, and how to avoid the common pitfalls that trip even experienced engineers.

### 1. Basic Usage

The `encoding/json` package relies on **struct tags** to control the mapping between Go fields and JSON names. Here’s a minimal, production‑ready example:

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "log"
    "time"
)

type User struct {
    ID        int64     `json:"id"`
    Name      string    `json:"name,omitempty"`      // Omit if zero value
    Email     string    `json:"email"`               // Always included
    CreatedAt time.Time `json:"created_at"`
    Password  string    `json:"-"`                   // Never serialized
}

func main() {
    u := User{
        ID:        42,
        Name:      "",
        Email:     "alice@example.com",
        CreatedAt: time.Now(),
        Password:  "secret",
    }

    // Marshal (struct → JSON)
    data, err := json.Marshal(u)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(string(data))
    // {"id":42,"email":"alice@example.com","created_at":"2025-03-15T10:30:00Z"}

    // MarshalIndent for human readability
    pretty, _ := json.MarshalIndent(u, "", "  ")
    fmt.Println(string(pretty))

    // Unmarshal (JSON → struct)
    jsonStr := `{"id":100,"email":"bob@example.com","created_at":"2025-03-15T11:00:00Z"}`
    var user User
    if err := json.Unmarshal([]byte(jsonStr), &user); err != nil {
        log.Fatal(err)
    }
    fmt.Printf("%+v\n", user)
    // {ID:100 Name: Email:bob@example.com CreatedAt:2025-03-15 11:00:00 +0000 UTC Password:}
}
```

**Streaming** – For large JSON values, use `json.Encoder` and `json.Decoder` to avoid loading the entire payload into memory:

```go
func processStream(r io.Reader) error {
    dec := json.NewDecoder(r)
    for dec.More() {
        var user User
        if err := dec.Decode(&user); err != nil {
            return err
        }
        // process user
    }
    return nil
}
```

### 2. Under the Hood

The `encoding/json` package is built on **reflection** (`reflect` package). At runtime, when you call `json.Marshal(v)`, the following happens:

1. **Type inspection** – The encoder walks the value’s type tree recursively. For structs, it retrieves the list of fields, parses the `json` tag, and caches this metadata per type.
2. **Caching** – The first time a type is marshaled, the encoder builds a **field‑encoding plan** (a slice of struct field metadata, including name, tag options, and a pointer to the field’s encoder function). This plan is stored in a `sync.Map` keyed by the type. Subsequent marshals reuse the plan, avoiding repeated tag parsing.
3. **Encoding dispatch** – For each field, a pre‑computed encoder function writes the JSON representation (e.g., `intEncoder`, `stringEncoder`, `structEncoder`). Primitive encoders are fast; struct encoders loop over their fields.
4. **Buffering** – The output is written to a `bytes.Buffer` (or directly to an `io.Writer` when using `json.Encoder`). The buffer grows dynamically; large allocations can be mitigated with `sync.Pool`.

**Decoder internals** are similar but more complex because JSON is a streaming, self‑describing format. The decoder tokenizes the input (e.g., `Delim('{')`, `String("id")`, `Number("42")`) and uses the same reflection‑cached plan to assign values into the target struct.

**Key insight:** The reflection‑based approach gives you “zero‑code” serialization at the cost of runtime type inspection. The cache makes subsequent marshaling almost as fast as hand‑written code, but the first call for a given type is noticeably slower (often 2–3×).

### 3. Why This Design?

The Go team chose **reflection over code generation** for the standard library’s JSON package because it aligns with Go’s core values:

- **Simplicity for the common case** – You don’t need a separate build step, derive macros (Rust), or annotation processors (Java). Write a struct, add tags, and it just works.
- **No magic build tools** – Go’s `go build` doesn’t require pre‑processing. The toolchain remains minimal.
- **Ease of learning** – A new Go developer can serialize data within minutes without understanding code generation or complex type mappers.

**The trade‑off** – Reflection is slower than compile‑time code generation and sacrifices some type safety (e.g., marshaling a struct with channels or functions will panic at runtime, not compile time). For teams that need maximum throughput, the standard library offers `json.Marshaler` and `json.Unmarshaler` interfaces, allowing you to bypass reflection entirely for hot paths.

**Why struct tags?** – Go does not have annotations or attributes in the Java/C# sense. Struct tags are a lightweight, **string‑based** mechanism that lives in the type system without adding new language syntax. They are parsed lazily by packages like `encoding/json`, `yaml`, `xml`, `bson`, and even `validation`. The downside: tags are not type‑checked, so misspellings (e.g., `json:”omitempty”` vs `json:”omitempty,omitempty”`) fail silently.

### 4. Competing Approaches

| Language / Library   | Approach                                   | Trade‑offs                                                                 |
|----------------------|--------------------------------------------|----------------------------------------------------------------------------|
| **Python (stdlib)**  | `json.dumps(dict)` – duck‑typed dicts      | No static types → runtime key errors. Fast but error‑prone.                |
| **Java (Jackson)**   | Annotations + reflection + code generation | Very powerful, but configuration‑heavy. Build‑time APT improves speed.    |
| **Rust (Serde)**     | Derive macros → compile‑time code gen      | Zero‑cost abstraction; blazing fast. Requires a build dependency & macro. |
| **C++ (nlohmann/json)** | Header‑only, operator overloading       | Easy to use, but heavy compile times and template bloat.                   |
| **Go (stdlib)**      | Reflection + caching + optional interfaces | Simple, no extra tools; runtime overhead but good enough for most APIs.   |

**Go’s unique position** – Unlike Rust’s Serde (which generates specialized code for each struct), Go’s reflection approach keeps the binary size small and the compilation model trivial. Like Java’s Jackson, it pays a runtime cost, but Go’s faster reflection and aggressive caching make it competitive for the 99th percentile of use cases.

### 5. Common Mistakes

Even seasoned engineers fall into these traps:

#### 5.1 `omitempty` and Zero Values

`omitempty` omits a field if its value is the **zero value** (`false`, `0`, `""`, `nil`). This is often not what you want for:
- `time.Time` – `time.Time{}` (the zero value) serializes to `"0001-01-01T00:00:00Z"`, but `omitempty` will omit the field entirely, changing the API contract.
- Pointers – A pointer field with `omitempty` is omitted if the pointer is `nil`. If you need to distinguish `null` from “absent”, use a pointer without `omitempty`.

```go
type Event struct {
    Timestamp time.Time  `json:"timestamp,omitempty"` // BAD: omits valid zero time?
    Data      *string    `json:"data,omitempty"`      // nil → omitted, &"" → `""`
}
```

**Fix:** For `time.Time`, use a pointer or a custom type that overrides `MarshalJSON`. For sentinel values, avoid `omitempty` or accept the behavior.

#### 5.2 Embedded Structs and Field Name Collisions

Embedding a struct brings its fields to the outer struct. If an embedded and outer field share the same JSON name, the outer field wins – but the behavior is subtle:

```go
type Base struct {
    ID int `json:"id"`
}
type Extended struct {
    Base
    ID int `json:"id"` // outer ID shadows Base.ID silently
}
```

The JSON output will contain only one `"id"` field (the outer). Use explicit field names or avoid shadowing.

#### 5.3 JSON Numbers and `float64`

By default, `encoding/json` unmarshals JSON numbers into `float64` when the target is an `interface{}`. This loses precision for large integers (> 2⁵³). To preserve integers, unmarshal into `json.Number` (deprecated in favor of `any` + `Decoder.UseNumber()`) or into a concrete `int64`/`uint64` field.

```go
var data map[string]interface{}
dec := json.NewDecoder(strings.NewReader(`{"id":9007199254740993}`))
dec.UseNumber()          // use json.Number instead of float64
dec.Decode(&data)
id, _ := data["id"].(json.Number).Int64() // works correctly
```

#### 5.4 `nil` Slice vs Empty Slice

`nil` slices marshal to `null`, while empty (non‑nil) slices marshal to `[]`. This distinction matters for APIs expecting an array (not `null`). Initialize slices with `make([]T, 0)` if you need `[]`.

```go
var s1 []string   // nil → "null"
s2 := []string{}  // non‑nil empty → "[]"
```

#### 5.5 `json.RawMessage` Misuse

`json.RawMessage` delays decoding by keeping the raw JSON bytes. A common mistake is to unmarshal into `*json.RawMessage` (a pointer) and then reuse the same variable without resetting – the raw bytes accumulate. Prefer value `json.RawMessage` unless you need to represent “field not present” (then use `*RawMessage`).

### 6. Performance Considerations

| Operation               | Allocation count (approx) | Complexity                       |
|-------------------------|---------------------------|----------------------------------|
| `json.Marshal(small struct)` | 3–5 allocs (buffer, map, encoder) | O(n) fields, but constant overhead |
| `json.Unmarshal(small struct)` | similar, plus reflection cache | O(n) fields                      |
| `json.Encoder` streaming | 0 heap allocs if you reuse buffer | amortized O(n)                   |
| `json.Decoder` streaming | low, reuses token buffer   | amortized O(n)                   |

**Key optimizations:**

- **Reuse buffers** – `json.Encoder` internally uses a `bufio.Writer`. For high‑throughput scenarios, create an encoder once and call `Encode` repeatedly, optionally resetting the underlying writer.
- **Avoid `map[string]interface{}`** – Unmarshaling into an interface map is significantly slower than into a typed struct because the decoder must dynamically allocate map entries and values.
- **Pre‑allocate maps** – When unmarshaling into a map, pre‑allocate with `make(map[string]T, expectedSize)` to reduce rehashing.
- **Use `json.RawMessage` for lazy decoding** – If you only need to inspect a part of the JSON, store the rest as `RawMessage` and decode later (or not at all).

**Benchmark example** (typical numbers on Go 1.22):
```
BenchmarkMarshalStruct-8    	 5000000	       295 ns/op	      96 B/op	       2 allocs/op
BenchmarkMarshalMap-8       	 2000000	       780 ns/op	     320 B/op	       8 allocs/op
BenchmarkUnmarshalStruct-8  	 3000000	       450 ns/op	     112 B/op	       3 allocs/op
```

**When reflection becomes the bottleneck** – For services handling >50k JSON messages per second, consider:
- `jsoniter` – drop‑in replacement that uses code generation and faster reflection (not always faster on recent Go versions).
- `easyjson` – compile‑time code generation (no reflection). Often 2–3× faster than `encoding/json` but adds a build step.
- Hand‑written `MarshalJSON` for critical types.

### 7. Best Practices

1. **Define stable struct tags** – Treat JSON tags as API surface. Use snake_case (common for REST) or camelCase, but be consistent. Add `omitempty` only when you understand its zero‑value behavior.

2. **Validate unknown fields in production** – Use `decoder.DisallowUnknownFields()` to catch typos or mismatched API versions. This prevents silent data loss.

   ```go
   dec := json.NewDecoder(r)
   dec.DisallowUnknownFields()
   ```

3. **Use `time.Time` with RFC3339** – The default marshaling uses `time.RFC3339Nano`. Ensure your API clients can parse that. Avoid custom time formats unless required.

4. **Never panic in custom `MarshalJSON`** – Return an `error` instead. Panicking inside a marshaler will crash your process. Prefer `fmt.Errorf` to propagate meaningful errors.

5. **Avoid `interface{}` (or `any`) as a catch‑all** – Use `json.RawMessage` if you need partial decode, or define a struct with optional fields (pointers). The compiler can’t help with `any`.

6. **Set `json.Encoder`’s `SetIndent` only for debug endpoints** – Indented JSON uses more bandwidth and CPU. For production APIs, compact JSON is standard.

7. **Handle large numbers as strings** – If your JSON contains 64‑bit integers (or bigger), use the `string` tag option to marshal them as JSON strings, preserving precision:

   ```go
   type BigInt struct {
       Value uint64 `json:"value,string"`
   }
   // {"value":"18446744073709551615"}
   ```

8. **Use `json.Number` when you must unmarshal into `any`** – Enable `Decoder.UseNumber()` to avoid float64 rounding.

### 8. Examples

#### Example 1: Custom Marshaling for Hex‑Encoded Bytes

```go
type HexBytes []byte

func (h HexBytes) MarshalJSON() ([]byte, error) {
    hexStr := hex.EncodeToString(h)
    return json.Marshal(hexStr)
}

func (h *HexBytes) UnmarshalJSON(data []byte) error {
    var s string
    if err := json.Unmarshal(data, &s); err != nil {
        return err
    }
    b, err := hex.DecodeString(s)
    if err != nil {
        return err
    }
    *h = b
    return nil
}

// Usage
type Record struct {
    Hash HexBytes `json:"hash"`
}

func main() {
    r := Record{Hash: []byte{0xDE, 0xAD, 0xBE, 0xEF}}
    out, _ := json.Marshal(r)
    fmt.Println(string(out)) // {"hash":"deadbeef"}
}
```

#### Example 2: Partial Decoding with `json.RawMessage`

```go
type EventEnvelope struct {
    Type    string          `json:"type"`
    Payload json.RawMessage `json:"payload"` // delay decoding
}

func handleMessage(data []byte) error {
    var env EventEnvelope
    if err := json.Unmarshal(data, &env); err != nil {
        return err
    }
    switch env.Type {
    case "user_created":
        var user User
        if err := json.Unmarshal(env.Payload, &user); err != nil {
            return err
        }
        // handle user creation
    case "user_deleted":
        var id struct { ID int64 `json:"id"` }
        if err := json.Unmarshal(env.Payload, &id); err != nil {
            return err
        }
        // delete user
    }
    return nil
}
```

### 9. Summary & Exercises

**Summary**  
- `encoding/json` uses reflection and struct tags to provide zero‑boilerplate serialization.  
- Performance is excellent for most applications, thanks to type‑plan caching.  
- Pitfalls include `omitempty` with zero values, number precision, and embedded struct field collisions.  
- For high‑throughput systems, consider streaming (`Decoder`/`Encoder`) and optional code generation (`easyjson`).  
- Go’s design prioritizes simplicity over raw speed; the standard library is intentionally “good enough” for the vast majority of web services.

**Exercises**

1. **Streaming JSON filter** – Write a program that reads a large JSON array (e.g., `[{"id":1}, {"id":2}, ...]`) from `stdin`, filters elements where `id` is even, and writes a new JSON array to `stdout`. Use `json.Decoder` and `json.Encoder` without loading the whole array into memory.  
   *Hint: Read the opening delimiter `[`, then decode each element in a loop.*

2. **Version‑aware unmarshaling** – Design a `Person` struct that has evolved: version 1 used `full_name` (string), version 2 uses `first_name` and `last_name`. Implement a custom `UnmarshalJSON` that can read either version and populate a canonical `Person` struct.  
   *Challenge: Preserve unknown fields using `json.RawMessage` and a fallback map.*

3. **Benchmark battle** – Write a benchmark that marshals a struct with 10 fields (including nested structs) using `encoding/json`, `jsoniter`, and hand‑written `MarshalJSON`. Compare ns/op and allocs/op on Go 1.21+. Decide when it’s worth abandoning the standard library.  
   *Goal: Understand the real cost of reflection in your own workload.*
