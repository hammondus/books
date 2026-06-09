# Chapter 11: Strings, Runes, and Unicode

## 1. Basic Usage

In Go, a `string` is a **read-only slice of bytes** that holds UTF-8 encoded text. Unlike languages with separate character types, Go uses `rune` (an alias for `int32`) to represent a Unicode code point.

```go
package main

import (
    "fmt"
    "unicode/utf8"
)

func main() {
    // String literals are UTF-8 encoded
    var s string = "Hello, 世界"
    
    // Length returns raw bytes, NOT characters
    fmt.Println("Byte length:", len(s))          // 13 (5+1+1+6? Let's see: 'H'(1) 'e'(1) 'l'(1) 'l'(1) 'o'(1) ','(1) ' '(1) '世'(3) '界'(3) = 13)
    
    // Indexing returns a byte (byte value at position)
    fmt.Println("First byte:", s[0])             // 72 ('H')
    
    // Range loop iterates over runes, automatically decoding UTF-8
    fmt.Print("Runes: ")
    for i, r := range s {
        fmt.Printf("%d:%c ", i, r)               // 0:H 1:e 2:l 3:l 4:o 5:, 6:  7:世 10:界
    }
    fmt.Println()
    
    // Convert to []rune for character-by-character manipulation
    runes := []rune(s)
    fmt.Println("Rune count:", len(runes))       // 9 (H,e,l,l,o,,, ,世,界)
    fmt.Printf("Third rune: %c\n", runes[2])     // 'l'
    
    // Convert to []byte when you need mutable byte operations
    bytes := []byte(s)
    bytes[0] = 'h'                               // Modify copy
    fmt.Println("Modified string:", string(bytes)) // "hello, 世界"
    
    // Decode a single rune from a byte slice
    r, size := utf8.DecodeRune([]byte("Hello"))
    fmt.Printf("First rune: %c (takes %d bytes)\n", r, size)
}
```

**Key operations**:
- `len(s)` → byte count (O(1))
- `s[i]` → byte at index `i` (O(1), but panics on out-of-range)
- `for i, r := range s` → decode runes, O(n) scan
- `[]rune(s)` → allocates a new slice, O(n) decode
- `string(runes)` → encodes back to UTF-8, O(n) allocate
- `string(bytes)` → copies bytes (since strings are immutable)

## 2. Under the Hood

### String Header

A `string` is a tiny **two-word struct** (like a slice, but without capacity):

```go
// runtime/string.go (conceptual)
type stringStruct struct {
    str unsafe.Pointer   // pointer to underlying byte array
    len int              // length in bytes
}
```

This header is **16 bytes** on 64-bit architectures. The actual bytes live in **read-only memory** (typically in the `.rodata` section of the binary or the heap for dynamically created strings).

### Immutability and Sharing

Because strings are immutable, Go freely shares backing memory:

```go
s := "hello world"
sub := s[6:]   // "world" – shares the same backing array, no copy
```

`sub` points to a **different offset** within the same read-only byte array. This is zero-cost (no allocation, O(1)). However, slicing by bytes is **dangerous** with multi-byte characters:

```go
s := "世界"
sub := s[:2]   // First 2 bytes – this is INVALID UTF-8 (only half a character)
fmt.Println(sub) // Prints garbage or replacement character
```

### UTF-8 Encoding

Go stores all strings as **UTF-8** (variable-width encoding):
- ASCII characters (0-127) → 1 byte
- Latin, Greek, Cyrillic → 2 bytes
- CJK (Chinese, Japanese, Korean) → 3 bytes
- Emoji and rare characters → 4 bytes

The `rune` type holds a **Unicode code point** (0 to 0x10FFFF). When you iterate over a string with `range`, Go decodes UTF-8 on the fly:

```go
for i, r := range "世" {
    // i = 0, r = 0x4E16 (19990)  "世" code point
    // This loop runs ONCE (not 3 times)
}
```

The iteration is **stateful** – it remembers position across bytes. Internally, it calls `utf8.DecodeRuneInString` at each step.

### Memory Layout Example

```
String: "Go 语言"
Bytes:  [71, 111, 32, 230, 178, 128, 232, 168, 128]
         G    o   ' '  '语' (3 bytes)    '言' (3 bytes)

Runes:  ['G', 'o', ' ', '语', '言']
Indices: 0    1    2    3      4
```

## 3. Why This Design?

### Why UTF-8 as the Internal Representation?

Go chose **UTF-8** for strings because:

1. **Zero overhead for ASCII** – Most strings in system software are ASCII, and UTF-8 treats them as single bytes, identical to C strings.

2. **No encoding conversion** – Many languages (Java, C#, JavaScript) store strings as UTF-16 internally, requiring conversion when reading/writing files or network data (which are typically UTF-8). Go avoids this penalty.

3. **Compatibility with C APIs** – Since UTF-8 is null-terminated compatible for ASCII, passing strings to C libraries is straightforward (`cgo` can use `char*` directly).

4. **Compactness** – For most text, UTF-8 is smaller than UTF-16 (especially for English/ASCII-heavy content). The Go team prioritized memory and bandwidth efficiency.

### Why Not a Separate `char` Type?

Languages like C and Rust have `char` (single byte) while Java and C# have 16-bit `char`. Go deliberately **has no character type** – a code point is just a `rune` (an integer). This forces you to think about encoding explicitly:

- "A character" is ambiguous (code point vs. grapheme cluster vs. byte)
- By making strings **byte sequences with UTF-8 semantics**, Go encourages you to handle variable-width encoding correctly rather than assuming fixed-width.

### Why Immutable Strings?

Immutability enables:
- **Safe concurrent access** – No need for locks when sharing strings across goroutines.
- **Memory efficiency** – Substrings share backing storage without copying.
- **Hash consistency** – Once created, a string's hash (if computed) never changes, making them ideal for map keys.

Trade-off: Building strings via repeated concatenation requires allocations. Hence `strings.Builder`.

## 4. Competing Approaches

### Java / C# (UTF-16, mutable alternatives)
- **Internal**: UTF-16 (2 or 4 bytes per code unit). Surrogate pairs for characters > U+FFFF.
- **Immutability**: Strings are immutable, but `StringBuilder` provides mutable buffer.
- **Trade-off**: UTF-16 wastes memory for ASCII (2x), but provides O(1) indexing of code units (not code points). Java's `char` is 16 bits, which cannot hold emoji (needs two `char`s).
- **Go's edge**: No surrogate pair complexity; direct UTF-8 handling matches external data.

### Python (Flexible string representation)
- **Internal**: Flexible: ASCII (1 byte), UCS2 (2 bytes), or UCS4 (4 bytes) depending on max code point. As of Python 3.3, "Flexible String Representation" switches dynamically.
- **Immutability**: Strings are immutable.
- **Trade-off**: Python's representation adds complexity and per-string overhead. Indexing returns a single-character string (not a code point). Python 3 strings are Unicode-aware, but indexing is O(1) only because it uses fixed-width storage.
- **Go's edge**: Simpler design (one encoding), predictable performance, no per-string overhead variability.

### Rust (UTF-8 enforced, two string types)
- **Internal**: UTF-8 bytes (`String` heap-allocated, mutable; `&str` borrowed slice).
- **Immutability**: `String` is mutable (can push bytes, but must maintain UTF-8 validity).
- **Trade-off**: Rust enforces UTF-8 validity at construction (no invalid sequences). Indexing by byte requires explicit slicing with range checks. `char` is 32-bit (like Go's `rune`).
- **Go's edge**: Go's `range` automatically handles invalid UTF-8 (replaces with `U+FFFD`). Rust panics on invalid UTF-8 in debug builds. Go prioritizes robustness over strictness.

### C++ (`std::string`)
- **Internal**: Sequence of `char` (usually bytes, encoding unspecified).
- **Trade-off**: No built-in Unicode support – you need external libraries (ICU, UTF-CPP). `std::string` is mutable, and length is byte count. The programmer is responsible for encoding awareness.
- **Go's edge**: First-class Unicode in the language and standard library (`unicode/utf8`, `unicode` packages).

### JavaScript (UTF-16)
- **Internal**: UTF-16 code units (like Java). Strings are immutable.
- **Trade-off**: The `length` property returns code units, not characters. Emoji (surrogate pairs) return length 2. Indexing returns code units (may be half a character).
- **Go's edge**: Explicit `rune` iteration clarifies the difference between bytes and code points.

## 5. Common Mistakes

### Mistake 1: Using `len()` for Character Count

```go
s := "Hello, 世界"
fmt.Println(len(s))        // 13 – WRONG for character count
fmt.Println(utf8.RuneCountInString(s)) // 9 – CORRECT
```

**Fix**: Always use `utf8.RuneCountInString` or convert to `[]rune` if you need indexable characters.

### Mistake 2: Byte-Indexing into Multi-byte Characters

```go
s := "世界"
fmt.Println(s[0])          // 228 (first byte of '世') – NOT a valid character
fmt.Println(s[1])          // 184
```

**Fix**: Convert to `[]rune` first or use `range`:

```go
runes := []rune(s)
fmt.Printf("%c\n", runes[0]) // '世'

// Or iterate with range
for i, r := range s {
    if i == 0 {
        fmt.Printf("%c\n", r) // '世'
    }
}
```

### Mistake 3: Assuming `range` Indexes are Sequential

```go
s := "世"
for i, r := range s {
    fmt.Printf("%d %c\n", i, r) // Prints "0 世" (NOT 0,1,2)
}
```

The index is the **byte offset**, not a rune index. This trips developers expecting C# or Python behavior.

### Mistake 4: String Concatenation in Loops

```go
// O(n²) allocations
s := ""
for i := 0; i < 10000; i++ {
    s += "a"  // Each iteration allocates and copies the entire string
}
```

**Fix**: Use `strings.Builder`:

```go
var builder strings.Builder
builder.Grow(10000) // Preallocate to avoid reallocations
for i := 0; i < 10000; i++ {
    builder.WriteByte('a')
}
s := builder.String()
```

### Mistake 5: Breaking UTF-8 Boundaries with Byte Slicing

```go
s := "Hello, 世界"
truncated := s[:8]  // Cuts in the middle of '世' (byte 7 is part of 3-byte sequence)
fmt.Println(truncated) // Prints "Hello, �" (invalid UTF-8)
```

**Fix**: Use `utf8.ValidString` and slice by runes:

```go
func truncateString(s string, maxBytes int) string {
    if len(s) <= maxBytes {
        return s
    }
    // Trim to valid UTF-8 boundary
    for !utf8.ValidString(s[:maxBytes]) && maxBytes > 0 {
        maxBytes--
    }
    return s[:maxBytes]
}
```

### Mistake 6: Comparing Strings with Case Sensitivity Incorrectly

```go
s1 := "Hello"
s2 := "hello"
if s1 == s2 { // false – case-sensitive byte comparison
}
```

**Fix**: Use `strings.EqualFold` for Unicode case folding:

```go
if strings.EqualFold(s1, s2) { // true
}
```

### Mistake 7: Forgetting That `range` Replaces Invalid UTF-8

```go
invalid := string([]byte{0xFF, 0xFE}) // Invalid UTF-8
for _, r := range invalid {
    fmt.Printf("%U\n", r) // Prints U+FFFD (replacement character)
}
```

If you need to detect invalid sequences, use `utf8.DecodeRuneInString` and check if `r == utf8.RuneError`.

## 6. Performance Considerations

### Memory Allocations

| Operation | Allocation | Complexity |
|-----------|------------|-------------|
| `string + string` | New allocation (copy both) | O(n+m) |
| `string([]byte)` | Copy bytes to read-only memory | O(n) |
| `[]byte(string)` | Copy bytes to heap (unless compiler optimizes) | O(n) |
| `[]rune(string)` | Decode + allocate rune slice | O(n) |
| `string(runes)` | Encode + allocate byte slice | O(n) |
| Substring `s[a:b]` | No allocation (shares backing) | O(1) |
| `strings.Builder.String()` | One final allocation | O(1) after building |

### The `[]byte` to `string` Conversion Trap

Converting `[]byte` to `string` **always copies** because strings are immutable. This cost can dominate hot paths:

```go
// Bad: copying on every iteration
data := []byte{...}
for {
    s := string(data) // Copy
    process(s)
}

// Better: reuse the byte slice and convert once
var s string = string(data) // Copy once
for {
    process(s)
}
```

However, the compiler can sometimes optimize away the copy if it can prove the byte slice is never mutated. **Don't rely on this** – it's fragile.

### UTF-8 Decoding Cost

Iterating over runes with `range` is **O(n)** in bytes, but each step decodes one code point. For ASCII-only strings, it's cheap (1 byte per iteration). For multi-byte text, it's still O(n) but with more work.

```go
// Fast: byte iteration
for i := 0; i < len(s); i++ {
    b := s[i] // O(1) per byte
}

// Slower: rune iteration (decodes UTF-8)
for _, r := range s {
    _ = r // Decodes variable-width sequences
}
```

If you only need ASCII checks, validate with `utf8.ValidASCII` first, then treat as bytes.

### String Interning

Go does **not** automatically intern strings (unlike Java's `String.intern()`). Two identical string literals in the same compilation unit may share memory (compiler optimization), but dynamically created strings do not:

```go
s1 := "hello"
s2 := "hello"
// s1 and s2 MAY point to same memory (compiler optimization)
s3 := string([]byte("hello")) // New allocation
s4 := "hello"[:]               // Shares with literal if literal exists
```

Use `sync.Map` or a custom interning cache if you need to deduplicate many identical strings.

## 7. Best Practices

### Prefer `strings.Builder` for Concatenation

```go
func joinStrings(strs []string) string {
    var b strings.Builder
    b.Grow(estimateTotalLength(strs)) // Preallocate for efficiency
    for _, s := range strs {
        b.WriteString(s)
    }
    return b.String()
}
```

### Use `strings.Join` for Slice Concatenation

```go
// Clear and efficient
result := strings.Join(parts, ",")
```

### Iterate Correctly Over Unicode

```go
// For character-by-character processing
for _, r := range text {
    if unicode.IsLetter(r) {
        // ...
    }
}

// For byte-level operations (performance-critical ASCII)
if utf8.ValidASCII(text) {
    for i := 0; i < len(text); i++ {
        b := text[i] // Safe because ASCII
    }
}
```

### Normalize Unicode When Comparing

Unicode has multiple representations for the same character (e.g., "é" can be U+00E9 or U+0065 + U+0301). Use `golang.org/x/text/unicode/norm`:

```go
import "golang.org/x/text/unicode/norm"

func normalize(s string) string {
    return norm.NFC.String(s) // Normalization Form C (composed)
}

if normalize(s1) == normalize(s2) {
    // Safe comparison
}
```

### Validate UTF-8 When Accepting External Input

```go
func validateUTF8(data []byte) error {
    if !utf8.Valid(data) {
        return fmt.Errorf("invalid UTF-8 encoding")
    }
    return nil
}
```

### Use `strings.Cut` for Splitting (Go 1.18+)

```go
// Instead of strings.SplitN
email := "user@example.com"
local, domain, found := strings.Cut(email, "@")
if found {
    // local = "user", domain = "example.com"
}
```

### Prefer `string` over `[]byte` for Map Keys

Using `string` as a map key avoids an allocation if the key already exists as a string. Converting `[]byte` to a string for a map lookup **copies** the data unless the compiler optimizes it (which it sometimes does for `map[string]` lookup with `string(b)`).

```go
// Suboptimal
m := make(map[string]int)
key := []byte("foo")
m[string(key)] = 42 // Allocation

// Better: use string literal or ensure key is already string
```

## 8. Examples

### Example 1: Reverse a String (Rune-Aware)

```go
func reverseString(s string) string {
    runes := []rune(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    return string(runes)
}

func main() {
    fmt.Println(reverseString("Hello, 世界")) // "界世 ,olleH"
}
```

### Example 2: Truncate to N Characters (Not Bytes)

```go
func truncateToRunes(s string, maxRunes int) string {
    if utf8.RuneCountInString(s) <= maxRunes {
        return s
    }
    // Iterate to find the byte position of the maxRunes-th rune
    count := 0
    for i := range s {
        if count == maxRunes {
            return s[:i]
        }
        count++
    }
    return s
}

func main() {
    s := "Hello, 世界"
    fmt.Println(truncateToRunes(s, 7)) // "Hello, " (7 runes including space)
}
```

### Example 3: Efficient String to []byte Zero-Copy (Unsafe – Use Sparingly)

```go
import "unsafe"

// StringToBytes converts a string to a byte slice without copying.
// WARNING: The returned byte slice is read-only! Modifying it causes undefined behavior.
func StringToBytes(s string) []byte {
    return unsafe.Slice(unsafe.StringData(s), len(s))
}

// BytesToString converts a byte slice to a string without copying.
// WARNING: The byte slice must not be mutated after conversion.
func BytesToString(b []byte) string {
    return unsafe.String(unsafe.SliceData(b), len(b))
}

func main() {
    s := "immutable"
    b := StringToBytes(s)
    // b[0] = 'x' // PANIC: attempt to write to read-only memory
    fmt.Println(string(b)) // "immutable"
}
```

**Only use this pattern when:**
- You have proven allocation is a performance bottleneck via profiling.
- You guarantee the byte slice will not be mutated (or the string will not be used after mutation).
- You understand that the GC may not track the reference correctly (though `unsafe.String` and `unsafe.Slice` are relatively safe in Go 1.20+).

### Example 4: Counting Emoji and Grapheme Clusters

```go
import "golang.org/x/text/unicode/norm"

// CountGraphemes counts user-perceived characters (grapheme clusters)
func CountGraphemes(s string) int {
    var iter norm.Iter
    iter.InitString(norm.NFC, s)
    count := 0
    for !iter.Done() {
        iter.Next()
        count++
    }
    return count
}

func main() {
    // "man with beard" emoji (multiple code points)
    s := "👨‍🦰" // U+1F468 + U+200D + U+1F9B0
    fmt.Println("Runes:", len([]rune(s)))           // 3
    fmt.Println("Graphemes:", CountGraphemes(s))    // 1
}
```

## 9. Summary & Exercises

### Summary

- Go strings are **immutable byte slices** with **UTF-8** encoding.
- `len()` returns **bytes**, not characters. Use `utf8.RuneCountInString` or `[]rune` for code point counts.
- `range` over a string iterates **runes** (decoding UTF-8), with index being byte offset.
- Slicing strings by bytes (`s[1:4]`) can break UTF-8 boundaries – validate with `utf8.ValidString`.
- Use `strings.Builder` for efficient concatenation, `strings.Join` for slices.
- Convert between `string` and `[]byte` or `[]rune` only when necessary – each conversion allocates.
- For Unicode normalization, use `golang.org/x/text/unicode/norm`.
- **Aha! moment**: Strings are not character sequences – they are UTF-8 byte sequences with a convenient decoder. Understanding the distinction between **bytes**, **code points (runes)**, and **grapheme clusters** is the key to correct text handling in Go.

### Exercises

#### Exercise 1: Build a Thread-Safe String Interning Cache

Implement a cache that deduplicates strings to save memory. The cache should:
- Accept a string and return a canonical (interned) version.
- Be safe for concurrent use by multiple goroutines.
- Never grow unbounded – implement a max size with LRU eviction.
- Hint: Use `sync.RWMutex` or `sync.Map`. Benchmark the trade-offs.

```go
type StringInterner interface {
    Intern(s string) string
}
```

#### Exercise 2: UTF-8 Validation Streaming Reader

Implement an `io.Reader` that wraps another `io.Reader` and validates that all data read is valid UTF-8. If invalid UTF-8 is detected, return an error at the point of detection (don't buffer the entire stream). The reader should maintain state across calls to `Read` (since a valid UTF-8 sequence may span multiple reads).

```go
type UTF8ValidatorReader struct {
    underlying io.Reader
    // Add state for partial runes
}
```

#### Exercise 3: Rune-Aware Substring with Bounds Checking

Write a function `SubstringByRunes(s string, start, end int) (string, error)` that returns the substring from rune index `start` (inclusive) to `end` (exclusive). Return an error if indices are out of range or if the resulting byte slice would cut a rune (though that shouldn't happen with correct rune indexing). The function must not allocate a `[]rune` of the entire string (to avoid O(n) memory for long strings). Instead, iterate through the string to find the byte offsets.

**Bonus**: Add grapheme cluster awareness using `norm.Iter` so that `start` and `end` refer to grapheme cluster indices, not rune indices.
