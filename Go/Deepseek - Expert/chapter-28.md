## Chapter 28: JSON & Data Serialization

Go’s `encoding/json` package is one of the most widely used serialization libraries in the world. It embodies the Go philosophy: a small, well-defined set of mechanisms that compose elegantly, sufficient for the vast majority of data exchange tasks without requiring external schemas or code generation. This chapter moves beyond the basics and into the internals, performance, and design trade-offs that matter when you are building services that process millions of JSON messages. We will also look at the upcoming JSON v2 package and what it promises for the ecosystem.

---

### 1. Basic Usage

The two fundamental operations are **marshaling** (Go → JSON) and **unmarshaling** (JSON → Go). The `encoding/json` package uses struct tags to map JSON keys to struct fields, and it supports streaming via `Encoder` and `Decoder`.

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "log"
    "os"
)

type Transaction struct {
    ID        string  `json:"id"`
    Amount    float64 `json:"amount"`
    Currency  string  `json:"currency"`
    Timestamp int64   `json:"ts"`
    Notes     string  `json:"notes,omitempty"`
}

func main() {
    t := Transaction{
        ID:       "txn_123",
        Amount:   99.95,
        Currency: "USD",
        Timestamp: 1718123456,
    }

    // Marshal to a byte slice
    data, err := json.Marshal(t)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(string(data))

    // Unmarshal from a byte slice
    raw := `{"id":"txn_456","amount":200.50,"currency":"EUR","ts":1718123500}`
    var t2 Transaction
    if err := json.Unmarshal([]byte(raw), &t2); err != nil {
        log.Fatal(err)
    }
    fmt.Printf("%+v\n", t2)

    // Streaming encoder (to os.Stdout)
    enc := json.NewEncoder(os.Stdout)
    enc.SetIndent("", "  ")
    _ = enc.Encode(t)

    // Streaming decoder (from a reader)
    var t3 Transaction
    dec := json.NewDecoder(bytes.NewReader([]byte(raw)))
    if err := dec.Decode(&t3); err != nil {
        log.Fatal(err)
    }
}
```

**Struct tag options** control the serialisation behaviour:

- `json:"field_name"` – renames the key.
- `omitempty` – omits the field if it is the zero value for its type (false, 0, empty string, nil pointer, etc.).
- `-` – always omit the field.
- `string` – forces a number or boolean to be encoded as a JSON string.

For example:

```go
type Config struct {
    Port   int  `json:"port,string"`            // → "port": "8080"
    Debug  bool `json:"debug,omitempty"`        // omitted when false
    Secret string `json:"-"`                    // never serialised
}
```

The standard library also provides `json.RawMessage` for deferred or conditional decoding, and `json.Number` to avoid loss of precision when dealing with large numbers.

---

### 2. Under the Hood

The `encoding/json` package relies heavily on **reflection** at runtime. When you call `json.Marshal(v)`, the encoder walks the structure of `v` using the `reflect` package. It inspects each field, reads its `json` tag, and builds the corresponding JSON output. The process is:

1. **Reflect on the value** to obtain its `reflect.Type` and `reflect.Value`.
2. **Cache the type information** – the first time a type is encountered, its field metadata is computed and stored in a global map (protected by a mutex). Subsequent marshals reuse this cached info.
3. **Encode values** recursively: for structs, iterate over fields; for slices/maps, iterate elements; for basic types, format using `strconv`.
4. **Write to a buffer** – `json.Marshal` uses an internal `bytes.Buffer` while `json.NewEncoder` writes directly to the provided `io.Writer`.

The decoder follows a mirrored path: it reads tokens from the JSON input, uses the reflection-based cache to map JSON keys to struct fields, and populates the target value.

**Memory and allocation**:

- `json.Marshal` allocates a new `[]byte` for every call. For high‑throughput paths this creates significant GC pressure.
- `json.NewEncoder` can reuse an internal buffer if you use `sync.Pool`, but by default it also allocates on each `Encode`. You can wrap the `io.Writer` to provide your own buffer pool.
- The reflection machinery itself causes allocations for `interface{}` boxing, but the package has been heavily optimised over many releases.

**Streaming behaviour**:

`json.Decoder` reads one complete JSON value from the stream. It buffers input internally and can handle multiple JSON objects in a stream (e.g., newline‑delimited JSON). `Decoder.DisallowUnknownFields()` instructs the decoder to return an error if the JSON contains a key not present in the target struct – a crucial option for robustness.

**JSON v2 – a new engine**:

Since Go 1.24, the `encoding/json/v2` package has been available as an experimental, drop‑in replacement. Its internals are radically different: it avoids reflection for types it can handle at compile time (using generics and code generation hooks), relies on a new tokenizer and encoder API that can operate with zero allocations for simple paths, and provides explicit options for handling duplicate keys, case‑insensitive matching, and number parsing. The v2 package is designed so that existing struct tag semantics are preserved, but performance‑critical code can opt into a lower‑allocation path without sacrificing the convenience of implicit struct mapping.

---

### 3. Why This Design?

The Go team’s decisions around JSON serialisation centre on three principles:

1. **Implicit satisfaction of serialisation interfaces**. If your type implements `json.Marshaler` or `json.Unmarshaler`, the encoder/decoder will use it. There are no mandatory base classes or annotations; you opt in by simply adding the method. This is the same “duck typing” approach that runs through all Go abstractions.

2. **Struct tags over external schema**. Instead of requiring an IDL file (like Protobuf) or a separate “binding” class (like Jackson’s annotations in Java), Go embeds the mapping directly in the source code. This keeps the data definition close to the logic, reduces the number of moving parts, and makes the serialisation behaviour visible at a glance. The cost is that the mapping is not verified until runtime, but in practice this rarely causes issues because unit tests quickly catch mismatches.

3. **Reflection over code generation by default**. Code generation (e.g., `easyjson`, `ffjson`) can produce faster serialisation, but it adds a build step and generates large amounts of repetitive code. For the vast majority of applications, the reflection‑based approach is fast enough and keeps the toolchain simple. The standard library is the “do the simple thing first” path; when you outgrow it, you can replace it with a code‑generated solution without changing your model types.

These choices reflect the broader Go philosophy: **Simplicity over Complexity**. A newcomer can marshal a struct to JSON in three lines, and that same code can serve a production service handling moderate traffic. When traffic becomes extreme, the escape hatches (`MarshalJSON`, custom `io.Writer` wrapping, the v2 package) are there without forcing everyone to pay the complexity cost upfront.

---

### 4. Competing Approaches

It is instructive to compare Go’s model with how other ecosystems handle JSON serialisation.

**Python (standard `json` module)**:
Python’s `json.dumps` and `json.loads` work on dictionaries. There is no static mapping; you parse JSON into dynamic `dict`/`list` structures and then manually extract fields. Type safety relies on optional type hints and external libraries like `pydantic`. Go’s approach provides compile‑time safety for the shape of the data once you define the struct, but it sacrifices the ad‑hoc flexibility that Python offers. For services with a well‑known schema, Go’s struct‑based approach eliminates a whole class of `KeyError` bugs.

**Java (Jackson / Gson)**:
Jackson uses annotations (`@JsonProperty`, `@JsonIgnore`) that are conceptually similar to struct tags, but because Java lacks structs, you must write a full class with getters/setters or use a `record`. Jackson can work via reflection or via a “streaming” API. The difference is that Jackson requires you to explicitly enable modules (e.g., for Java 8 time) and often demands configuration objects. Go’s implicit `Marshaler`/`Unmarshaler` interfaces feel lighter—they are just method definitions, not library‑specific annotations.

**C++ (nlohmann/json, RapidJSON)**:
C++ libraries tend to be either DOM‑based (parse into a generic `json` object) or SAX‑based (event callbacks). Mapping to user‑defined structs typically requires manual `to_json`/`from_json` functions or macros. C++20’s reflection is still maturing. Go’s struct tags give a more integrated experience without macros or external pre‑processors.

**Rust (serde)**:
Serde is the gold standard for zero‑cost serialisation. It uses derive macros to generate serialisation code at compile time. The result is extremely fast, with no runtime reflection overhead. Rust’s ownership model allows serde to avoid unnecessary copies. Go’s reflection incurs a runtime cost that serde avoids, but at the expense of a mandatory code‑generation step (the derive macros). Go’s `json/v2` package closes this gap by offering an opt‑in code‑generation mode, acknowledging that high‑throughput systems can justify the extra tooling.

**Summary**: Go chooses **simplicity of setup** over peak performance. For most line‑of‑business applications, the reflection‑based encoder is perfectly adequate. When it isn’t, the language provides well‑defined optimisation paths without forcing you to abandon the standard library.

---

### 5. Common Mistakes

Even experienced engineers can stumble when moving from other languages to Go’s JSON handling.

**Unexported fields are silently ignored.** This is the most frequent gotcha. The `encoding/json` package can only see exported fields (those starting with a capital letter). An unexported field with a matching json tag will not be populated and will not cause an error.

```go
type user struct {
    Name string `json:"name"` // never filled
}
```

**`omitempty` and zero‑value structs.** A struct field is considered empty if it is its zero value. For `time.Time`, the zero value is `time.Time{}` (year 1, month 1, etc.). That is never considered “empty” by `omitempty`, so it will always be included. To truly omit a zero time, use a pointer: `*time.Time`.

**`json.Number` vs. `float64`.** By default, JSON numbers are decoded into `float64`, which can lose precision for integers above 2⁵³. If your JSON contains large identifiers (e.g., a bank transaction ID like `9007199254740993`), decode with `Decoder.UseNumber()` and then convert with `json.Number.Int64()`.

**Forgetting to check for unknown fields.** If a client sends a misspelled key (e.g., `"amoutn"` instead of `"amount"`), the standard decoder silently ignores it. In production, this hides bugs. Always use `dec.DisallowUnknownFields()` for request parsing.

**Double‑encoding with `json.RawMessage`.** `json.RawMessage` is already a valid JSON literal. If you put a `RawMessage` inside a struct and then marshal the struct, the encoder will treat the `RawMessage` as a byte slice and re‑encode it, escaping quotes. The correct pattern is to use a custom marshaler that writes the raw bytes directly, or to structure your code so that the `RawMessage` is assigned to a field of type `json.RawMessage` – the encoder automatically handles it properly (it writes the raw bytes without escaping).

**Assuming map ordering.** JSON objects are unordered. If you unmarshal into a `map[string]any`, the iteration order over that map is non‑deterministic, just like any Go map. Do not rely on key ordering.

**Blocking on huge payloads with `Unmarshal`.** `json.Unmarshal` reads the entire payload into memory. For multi‑megabyte JSON arrays, use `json.Decoder` in streaming mode to process elements one by one.

---

### 6. Performance Considerations

When you push beyond a few thousand messages per second, the cost of `encoding/json` becomes measurable.

**Allocation counts**: Marshaling a typical struct with a dozen fields can allocate 10–20 objects, mainly due to reflection and the internal byte buffer growing. Each allocation adds GC pressure. Using `Encoder` with a pooled writer can halve the number of allocations because the final byte slice is not separately allocated if you write directly to a reusable buffer.

**CPU overhead**: The reflection path involves numerous checks, slice bounds, and interface conversions. For a struct that is known at compile time, a hand‑written `MarshalJSON` method that writes directly to a `[]byte` buffer can be 5–10× faster. Code‑generation tools like `easyjson` achieve similar speed by generating the exact field‑by‑field encoding logic.

**Streaming vs. buffering**: For very large responses (e.g., streaming a large dataset to an HTTP client), `json.NewEncoder(w)` is more memory‑efficient than marshaling to a `[]byte` and then writing. The encoder writes JSON tokens incrementally, keeping only a small internal buffer.

**The v2 advantage**: The experimental `encoding/json/v2` package introduces a “fast path” that uses a new interface, `jsonv2.MarshalerV2`, which operates on a `jsonv2.Encoder` directly, completely bypassing reflection. Even without custom implementations, the v2 package has a redesigned tokenizer that reduces allocations by an order of magnitude for many workloads. Early benchmarks show 2–4× throughput improvement for typical struct serialisation.

**GC‑friendly patterns**:
- Reuse `json.Encoder` instances by wrapping an `io.Writer` from a pool.
- Pre‑allocate byte slices with a sensible initial capacity when building custom marshalers.
- Avoid decoding into `map[string]any` when you can use a struct; the map incurs many small allocations for keys and values.

**Big O complexity**: Marshaling is O(n) where n is the total number of values in the data structure. Reflection adds a constant factor but does not change the asymptotic complexity. Streaming decoding of a JSON array of unknown length is also O(n), but you can process elements as they arrive, keeping memory usage constant.

---

### 7. Best Practices

**1. Use strict decoding for external input.**
Every `Decoder` that reads from a network should call `DisallowUnknownFields()`. This catches client errors immediately.

```go
dec := json.NewDecoder(r.Body)
dec.DisallowUnknownFields()
if err := dec.Decode(&req); err != nil {
    http.Error(w, "invalid request: "+err.Error(), http.StatusBadRequest)
    return
}
```

**2. Prefer structs over `map[string]any` for known schemas.**
Structs provide type safety, documentation, and better performance. Reserve `map[string]any` for truly dynamic JSON (e.g., user‑defined metadata fields).

**3. Handle optional fields with pointers.**
An optional field that can be `null` or absent is best modelled as a pointer. A `*string` distinguishes between `nil` (absent) and `""` (present but empty). With `omitempty`, a `nil` pointer is omitted, an empty string pointer with `*s = ""` is serialised as `"field": ""`.

**4. Implement custom marshalers for non‑trivial types.**
For `time.Time`, prefer a custom layout:

```go
type RFC3339Time time.Time

func (t RFC3339Time) MarshalJSON() ([]byte, error) {
    return json.Marshal(time.Time(t).Format(time.RFC3339))
}
```

**5. Use `json.RawMessage` for deferred decoding.**
When you need to decide the concrete type based on a discriminator field, first unmarshal into a struct that captures the discriminator and the rest as `json.RawMessage`. Then unmarshal the raw message into the appropriate type.

**6. Exhaust every resource when streaming.**
Close the source reader or ensure the entire stream is consumed. A `json.Decoder` may buffer data; leaving the underlying connection unread can cause resource leaks.

**7. Keep tags consistent and minimal.**
Use `json:"name"` only when the JSON key differs from the (lowercase) Go field name. Every exported field gets a key equal to its name by default. Over‑tagging adds visual noise.

**8. Consider the v2 package for new high‑throughput services.**
While still experimental, `encoding/json/v2` is stable enough for early adopters. It is designed to be largely a drop‑in replacement, and it provides a clear path for incremental optimisation.

---

### 8. Examples

**Example 1: Custom marshaler that redacts a sensitive field**

```go
type User struct {
    Name  string `json:"name"`
    Email string `json:"email,omitempty"`
    SSN   string `json:"-"` // never serialised
}

func (u User) MarshalJSON() ([]byte, error) {
    type Alias User // prevent recursion
    return json.Marshal(&struct {
        *Alias
        SSN string `json:"ssn"`
    }{
        Alias: (*Alias)(&u),
        SSN:   "***-**-****",
    })
}
```

**Example 2: Streaming an HTTP response with pooled buffers**

```go
var bufPool = sync.Pool{
    New: func() any {
        return &bytes.Buffer{}
    },
}

func handler(w http.ResponseWriter, items []Item) {
    buf := bufPool.Get().(*bytes.Buffer)
    defer bufPool.Put(buf)
    buf.Reset()

    enc := json.NewEncoder(buf)
    for _, item := range items {
        if err := enc.Encode(item); err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }
    }
    w.Header().Set("Content-Type", "application/x-ndjson")
    _, _ = w.Write(buf.Bytes())
}
```

**Example 3: Polymorphic decoding with discriminator**

```go
type Message struct {
    Type    string          `json:"type"`
    Payload json.RawMessage `json:"payload"`
}

type TextPayload struct {
    Body string `json:"body"`
}
type ImagePayload struct {
    URL string `json:"url"`
}

func decodeMessage(data []byte) (any, error) {
    var msg Message
    if err := json.Unmarshal(data, &msg); err != nil {
        return nil, err
    }
    switch msg.Type {
    case "text":
        var t TextPayload
        if err := json.Unmarshal(msg.Payload, &t); err != nil {
            return nil, err
        }
        return t, nil
    case "image":
        var i ImagePayload
        if err := json.Unmarshal(msg.Payload, &i); err != nil {
            return nil, err
        }
        return i, nil
    default:
        return nil, fmt.Errorf("unknown type: %s", msg.Type)
    }
}
```

**Example 4: JSON v2 fast‑path (experimental)**

```go
//go:build go1.24
// +build go1.24

package main

import (
    "fmt"
    jsonv2 "encoding/json/v2"
)

type Point struct {
    X, Y float64
}

func (p Point) MarshalJSONV2(enc *jsonv2.Encoder) error {
    enc.WriteObjectStart()
    enc.WriteString("x")
    enc.WriteFloat64(p.X)
    enc.WriteString("y")
    enc.WriteFloat64(p.Y)
    enc.WriteObjectEnd()
    return nil
}

func main() {
    p := Point{X: 3.14, Y: 2.71}
    data, err := jsonv2.Marshal(p)
    if err != nil {
        panic(err)
    }
    fmt.Println(string(data))
}
```

---

### 9. Summary & Exercises

This chapter unpacked the machinery behind `encoding/json`: its reflection‑based engine, the philosophy of implicit interfaces and struct tags, and how to sidestep common pitfalls that manifest under load. You now understand the trade‑offs between the default “just works” approach and the explicit optimisation paths, including the upcoming `json/v2` package.

**Key takeaways**:
- Struct tags and reflection provide a low‑ceremony mapping that covers the majority of use cases.
- The streaming `Encoder`/`Decoder` are essential for large payloads and network I/O.
- Always use `DisallowUnknownFields()` for untrusted input.
- Pointers are the idiomatic way to model nullable/optional fields.
- Performance‑critical paths benefit from pooled buffers, manual marshalers, or code generation, and the v2 package will further reduce the cost of these optimisations.

**Exercises**:

1. **Thread‑safe JSON logger.** Build a type `LineLogger` that accepts arbitrary structs, marshals each to a single JSON line, and writes them to an `io.Writer` from multiple goroutines safely. Use a buffered channel to serialise writes and ensure no partial lines. Add a graceful shutdown mechanism that flushes all pending lines.

2. **Polymorphic message router.** Implement a `MessageHandler` that reads from an `io.Reader` a stream of JSON objects (one per line), each containing a `"type"` field. Based on the type, unmarshal the object into a concrete Go type and dispatch it to a registered callback. The router must reject unknown fields and log malformed messages without stopping the stream. Use `json.RawMessage` for deferred decoding.

3. **Custom `time.Time` marshaler with omitempty.** Go’s `time.Time` zero value is `0001-01-01T00:00:00Z`. Design a wrapper `NullableTime` that, when marshaled, omits itself entirely if the time is the zero value, but otherwise serialises in RFC 3339 format. Ensure that unmarshaling `null` or an absent key results in the zero value. Provide a method `IsSet()` for callers to check presence. Write a table‑driven test that validates the serialisation and deserialisation behaviour.
