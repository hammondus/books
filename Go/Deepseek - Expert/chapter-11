## Chapter 11: Strings, Runes, and Unicode

In Go, a string is an immutable sequence of bytes. That sentence alone contains three critical design choices—immutability, byte orientation, and the absence of a dedicated “character” type—that shape every string manipulation you will ever write. If you come from a language where a string is an array of `char` (Java, C#) or a sequence of Unicode code points (Python 3), Go’s model can feel primitive at first glance. But its primitiveness is deliberate: by exposing bytes and forcing you to think about encoding, Go gives you precision, performance, and freedom from hidden conversion costs. This chapter dives into the string-rune-UTF-8 triumvirate, explaining what happens in memory, why the design is the way it is, and how to avoid the traps that ensnare developers who treat strings as just “text.”

### 1. Basic Usage

A string literal is a sequence of bytes enclosed in double quotes. Backticks create raw string literals where backslashes have no special meaning—ideal for regex or multi-line content.

```go
s := "Hello, 世界"
raw := `line1
line2`
```

The built-in `len()` returns the number of **bytes**, not characters.

```go
fmt.Println(len(s)) // 13: 7 ASCII bytes + 2×3 bytes for each Chinese character
```

To count characters—more precisely, Unicode code points—use `utf8.RuneCountInString` or convert to `[]rune` and take its length.

```go
import "unicode/utf8"

fmt.Println(utf8.RuneCountInString(s)) // 9
fmt.Println(len([]rune(s)))            // 9, but allocates
```

Iteration over a string with `range` yields **runes** (code points) and their starting byte index:

```go
for i, r := range s {
    fmt.Printf("byte offset %d: %c\n", i, r)
}
// byte offset 0: H
// byte offset 1: e
// ...
// byte offset 7: 世
// byte offset 10: 界
```

Notice the index jumps by the rune’s byte length. If you need to iterate by bytes, use a standard `for` loop with `s[i]`, but that gives individual bytes, not characters.

A rune is simply an alias for `int32`. You can convert a rune to a string:

```go
var r rune = '界' // rune literal with single quotes
str := string(r)  // "界"
```

Conversions between `string`, `[]byte`, and `[]rune` are explicit and always **copy** the data:

```go
b := []byte(s)  // allocates new byte slice, copies
rs := []rune(s) // allocates new rune slice, copies
s2 := string(b) // allocates new string, copies
```

To extract a substring, use slicing. **Caution:** slicing operates on bytes, not runes. Slicing in the middle of a multi-byte rune produces a string with an invalid UTF-8 tail or head.

```go
s := "café"
prefix := s[:3] // "caf" — okay, 3 bytes happen to be valid
broken := s[:2] // "ca" — okay, 2 bytes are ASCII
// But if you slice differently:
s2 := "café"
sub := s2[:3] // "caf" — fine because 'é' is two bytes; 3 bytes still yields "caf"
```

We’ll explore the consequences and remedies later.

### 2. Under the Hood

At runtime, a string is represented by a small struct—`reflect.StringHeader`—consisting of a pointer to the underlying byte array and a length (in bytes). There is no capacity field because a string is immutable; you cannot `append` to it.

```go
// runtime/string.go (conceptual)
type stringStruct struct {
    str unsafe.Pointer
    len int
}
```

The data is just a contiguous block of bytes. There is no encoding metadata. A string **always** holds valid UTF-8 by convention, but the runtime does not enforce this. You can construct a string from arbitrary bytes, even invalid UTF-8.

```go
b := []byte{0xff, 0xfe, 0xfd}
s := string(b) // perfectly legal, contains garbage as UTF-8
fmt.Println(utf8.ValidString(s)) // false
```

This is a key insight: Go’s strings are byte sequences with a strong cultural expectation of UTF-8, not a mandate.

When you write a string literal in source code, the Go compiler encodes it as UTF-8 and stores the byte sequence in the binary’s read-only data segment. String literals are interned at compile time, but dynamically created strings are not automatically deduplicated (though the compiler may merge identical literals within the same package).

**Immutability** means any operation that appears to modify a string—concatenation, slicing, conversion—creates a new string or a new view. Slicing a string does not copy the bytes; it creates a new `StringHeader` with a possibly offset pointer and a new length. This is efficient but can keep large underlying arrays alive if you hold a tiny substring.

```go
bigString := strings.Repeat("x", 1<<20) // 1 MiB
tiny := bigString[:5]                    // just a new header, no copy
// bigString’s backing array remains in memory as long as tiny is reachable
```

The `range` loop over a string is implemented as a state machine that decodes UTF-8 on the fly. The compiler generates calls to `runtime.decoderune` which processes bytes without allocating. This makes `for range` both convenient and efficient—it never allocates just to iterate.

Converting a `string` to `[]byte` calls `runtime.stringtoslicebyte`, which allocates a new byte slice and copies the data. The reverse, `[]byte` to `string`, allocates a new string and copies. The only case that avoids a copy is the unsafe `unsafe.Slice` trick (which we’ll touch on in Performance), but that bypasses immutability guarantees and should be used only when profiling proves it’s necessary.

A `[]rune` conversion decodes the entire string into a slice of `int32`. This decodes all UTF-8 sequences at once, allocating a slice of length equal to the number of runes.

### 3. Why This Design?

Go’s string design is a deliberate “less is more” choice. The Go team could have given us a Unicode-aware string type with a native character type, indexed by code point, with automatic normalization. Instead, they gave us immutable byte slices and a library. Why?

**Simplicity and transparency.** A string is just bytes. There is no hidden encoding, no variable internal representation (like Python 3’s flexible storage or Java’s UTF-16). You always know exactly what you pay for: length in bytes, iteration cost, memory footprint. There is no expensive hidden conversion when crossing an API boundary; a string is exactly the bytes it contains.

**UTF-8 as the source of truth.** Ken Thompson and Rob Pike designed UTF-8 itself, so it was natural to make it the sole encoding of Go source code and the preferred encoding for strings. UTF-8 is self-synchronizing, ASCII-compatible, and space-efficient for Western text. It avoids the endianness problems of UTF-16 and the bloat of UTF-32. By baking it into the language only as a convention, Go encourages you to “just use UTF-8” without imposing it on arbitrary binary data.

**Immutability** provides safety in a concurrent world. Strings can be shared across goroutines without locks. It also allows the compiler to optimize memory by pointing multiple string variables at the same backing array (as with slicing). Immutable strings simplify reasoning about data flow: functions you call cannot modify your string.

**No character type.** Why `rune` is just `int32` rather than a separate, type-safe `char`? Because a character type would inevitably require encoding/decoding at I/O boundaries, and Go prefers to keep I/O as bytes. A `rune` is a Unicode code point—an integer—and that’s enough. It also sidesteps the temptation to index strings by characters, which would be an O(1) illusion masking O(n) reality. By making `s[i]` a byte, Go pushes the cost of rune access to the surface.

**Composition over inheritance.** Strings don’t inherit from anything. They compose perfectly with byte slices and readers. The `io.Reader` interface works with `[]byte`, not `string`, encouraging streaming and efficient buffer reuse. The standard library provides everything else: `strings`, `bytes`, `strconv`, `unicode`, `unicode/utf8`, and `golang.org/x/text`.

This design forces you to confront encoding early, but once you internalize the byte/rune distinction, you gain a robust, predictable model that scales from microcontrollers to cloud services.

### 4. Competing Approaches

**Java / C#.** Both use immutable strings, but their internal encoding is UTF-16. A `char` is a 16-bit code unit, not a full code point. Characters outside the Basic Multilingual Plane require surrogate pairs, making indexing a `char` risky. Java’s `String.length()` returns the number of `char` units, not code points—just like Go’s `len()` returns bytes. To get actual code points, you use `codePointCount`. The underlying representation is hidden, so the cost of accessing a code point by index varies. Java’s `String` is an object with a backing `byte[]` (in modern JVMs compact strings are stored as Latin-1 when possible), but you never see that byte array directly. Go’s approach is more transparent: you always know you’re dealing with bytes, and you must explicitly decode. That transparency avoids surprise O(n) operations hidden behind familiar syntax.

**Python 3.** Python strings are sequences of Unicode code points. Internally, Python selects a representation (1, 2, or 4 bytes per code point) based on the highest code point in the string, making indexing O(1) true for characters. This is convenient but has hidden memory tradeoffs: a string of mostly ASCII with one emoji can expand to 4 bytes per character. Python’s strings are immutable and hashable, enabling dictionary keys, but they are not byte sequences. To work with bytes, you must use `bytes` or `bytearray`, and conversion between `str` and `bytes` requires an explicit encoding. Go unifies the two perspectives: a string is a readable, immutable byte sequence that you interpret as UTF-8 when needed. There’s no separate `bytes` literal type for strings. This reduces API surface and duplication.

**C++.** C++ provides `std::string`, which is a mutable sequence of `char` (bytes). It has no built-in UTF-8 awareness; iterators traverse bytes. C++11 added `char16_t`/`char32_t` and `u16string`/`u32string`, but they are not seamlessly interoperable. Working with Unicode in C++ often requires libraries like ICU. Go’s standard library gives you the essential building blocks (`unicode/utf8`, `strings`, `unicode`) without pulling in a massive ICU dependency. Go’s string is immutable, unlike `std::string`, which eliminates many categories of aliasing bugs and makes strings safe to share across goroutines.

**Rust.** Rust’s `String` is a mutable, UTF-8 encoded, growable buffer, and `&str` is a borrowed view of a UTF-8 byte slice. Rust **enforces** valid UTF-8: creating a `String` from arbitrary bytes returns a `Result`, and slicing must fall on character boundaries or you get a panic. This strictness prevents many Go “gotchas” but adds ceremony when dealing with arbitrary byte sequences. Rust also has a `char` type that is 4 bytes, representing a Unicode scalar value. Go’s approach is more relaxed: you can hold invalid UTF-8 in a string if you really want, which is useful when working with legacy encodings or binary data. The Rust compiler forces you to think about encoding correctness at all times; Go gives you the tools and trusts you to use them.

In all these comparisons, Go’s string design stands out for its clarity and minimalism. You’re never far from the bytes, and the cost model is always visible.

### 5. Common Mistakes

**Mistake 1: Using `len()` to get character count.**
`len(s)` returns bytes. Use `utf8.RuneCountInString(s)` or `len([]rune(s))` for the number of code points. This mistake routinely breaks UI character limits and validation logic when non-ASCII text is involved.

**Mistake 2: Indexing a string expecting a character.**
`s[i]` is a `byte`. For rune access, use `[]rune(s)[i]` (allocates) or decode manually with `utf8.DecodeRuneInString`. This is a frequent source of off-by-one bugs when porting code from languages with a `char` type.

**Mistake 3: Slicing a string in the middle of a multi-byte rune.**
```go
s := "café"      // bytes: 63 61 66 C3 A9
sub := s[:4]     // "caf" + half of 'é'? Actually 'é' is 2 bytes (C3 A9), so s[:4] = "caf\u00c3" (invalid)
```
The resulting string contains a leading byte without its continuation byte. If you pass this to a function that expects valid UTF-8, it may produce replacement characters or errors. Always slice at known rune boundaries, or use `utf8.DecodeRune` to find them.

**Mistake 4: Forgetting that `range` over a string skips invalid UTF-8.**
When `range` encounters an invalid byte, it produces `rune = 0xFFFD` (the Unicode replacement character) and advances one byte. This masks data corruption. If you need to detect invalid sequences, validate with `utf8.ValidString` first or use `utf8.DecodeRune` in a manual loop to handle errors explicitly.

**Mistake 5: Comparing strings from different sources without normalization.**
Unicode allows multiple representations of the same visual character. For example, "é" can be a single code point (U+00E9) or a combination of "e" (U+0065) and a combining acute accent (U+0301). Go’s `==` compares byte sequences, so these will not be equal. Use `golang.org/x/text/unicode/norm` to normalize before comparison, or `strings.EqualFold` for case‑insensitive matching.

**Mistake 6: Building large strings with `+` in a loop.**
Each `+` creates a new string, copying everything. This turns an O(n) operation into O(n²). Use `strings.Builder` or `bytes.Buffer`. (We’ll dissect this in Performance.)

**Mistake 7: Assuming `string([]byte{...})` avoids allocation.**
It always allocates and copies. Many developers coming from C or C++ expect a zero-cost cast. In Go, strings are immutable, so the copy is mandatory to guarantee that the original byte slice can’t mutate the string’s contents. We’ll see how to safely avoid this in Performance, but the default is “copy.”

**Mistake 8: Treating `[]rune` as cheap.**
Converting a string to `[]rune` decodes the entire string and allocates a new slice. Doing this frequently in hot paths can generate significant GC pressure. Prefer iterating with `range` or using the `utf8` package’s decoding functions.

### 6. Performance Considerations

All string performance centers on allocation, copying, and UTF-8 decoding costs.

- **`len(s)`** is O(1) and does not examine the bytes; the length is stored in the header.
- **`utf8.RuneCountInString(s)`** is O(n) in bytes; it walks the string decoding each rune. There is no cached rune count.
- **`s[i]`** is O(1), a simple memory read. But accessing the nth rune is O(n) because you must decode from the beginning (or from some offset). Go provides no random-access rune index in the standard library.
- **Slicing a string** is O(1) and creates no copy; it only creates a new header. Watch for memory retention: if you slice a tiny prefix from a huge string, the huge backing array remains alive.
- **`for range`** over a string decodes UTF-8 on the fly with zero allocations. It is the most efficient way to iterate runes.
- **String concatenation with `+`** in a loop is a classic performance pitfall. Each iteration allocates a new string and copies all accumulated data. The total work is quadratic. Use `strings.Builder`:
  ```go
  var b strings.Builder
  for _, part := range parts {
      b.WriteString(part)
  }
  result := b.String()
  ```
  `strings.Builder` minimizes allocations by maintaining a growable byte buffer. Its `Grow` method can preallocate if you know the final size.
- **`string([]byte)` and `[]byte(string)`** copy the data. For large buffers, this is expensive. In performance-critical code, prefer working directly with `[]byte` when mutability is acceptable, and convert to string only at the final boundary. If you *must* avoid the copy and you fully control the lifetime, you can use `unsafe.Slice`/`unsafe.String`:
  ```go
  func unsafeString(b []byte) string {
      return *(*string)(unsafe.Pointer(&b))
  }
  ```
  This is dangerous: the resulting string points to the same memory as the slice, so modifying the slice’s content (or appending to it and causing a reallocation) will corrupt the string or cause a crash. Only use this when the byte slice will never be mutated again, and document it heavily.
- **`[]rune(s)`** decodes the whole string and allocates a slice of `int32`. For short strings it’s fine; for long strings consider if you can accomplish your goal with `range` or `utf8.DecodeRuneInString`.
- **String comparison** with `==` is a byte-for-byte comparison. It’s fast for short strings and when prefixes differ. Normalization before comparison adds overhead; only do it when necessary.
- **String interning** is not built-in. If your application uses many duplicate long strings (e.g., keys in a cache), you can reduce memory by maintaining a `map[string]string` deduplication pool. Be mindful of lock contention if shared across goroutines.

The garbage collector scans strings as pointer-containing objects: the string header contains a pointer. A large, unused string backing array will be collected only when no string header references it. The slicing trap can cause memory leaks: keeping a small substring of a huge string pins the entire large array. To break the reference, explicitly copy the substring: `string([]byte(bigString[:5]))`.

### 7. Best Practices

**Iterate runes with `for range`.** It’s concise, non-allocating, and handles invalid UTF-8 gracefully (though you may want to validate).

**Get the rune count with `utf8.RuneCountInString`** if you need it. Avoid `len([]rune(s))` solely for counting; it wastes memory.

**When slicing, ensure rune boundaries.** Use `utf8.DecodeRuneInString` to find boundary offsets, or use the `strings` package’s `Split`/`Index` functions which are byte‑based but safe if you search for ASCII delimiters. If you must split at an arbitrary index, first iterate runes and record byte positions.

**Normalize externally sourced text** before storage or comparison. The `golang.org/x/text/unicode/norm` package provides NFC, NFD, NFKC, NFKD forms. NFC is a good default for general text processing. Apply normalization at entry points (HTTP handlers, file readers) and store normalized strings.

**Use `strings.Builder` for dynamic string construction.** It’s the idiomatic, performant way. For fixed-number concatenations (e.g., 2-3 parts), `+` or `fmt.Sprintf` is acceptable and readable.

**Treat strings as immutable byte sequences.** If you need to manipulate bytes, use `[]byte` and convert to string at the end. This reduces the number of copies.

**Validate UTF-8 when correctness matters.** Use `utf8.ValidString(s)` before passing strings to systems that require valid UTF-8. For data streams, `utf8.Valid` on byte slices.

**Be deliberate about memory.** If you hold a small substring from a large source and the large source is no longer needed, copy the substring to free the backing array:

```go
small := string([]byte(large[n:m])) // explicit copy
```

**Prefer `strings.Index` over `bytes.Index` for string search.** The `strings` functions are optimized and don’t require conversion.

**Use `string` type for text, `[]byte` for mutable buffers or I/O.** Avoid converting back and forth unnecessarily.

### 8. Examples

**Truncate a string to max runes without breaking characters:**

```go
func truncateByRunes(s string, maxRunes int) string {
    if maxRunes <= 0 {
        return ""
    }
    count := 0
    for i := range s {
        if count == maxRunes {
            return s[:i]
        }
        count++
    }
    return s
}
// Usage:
fmt.Println(truncateByRunes("Hello, 世界!", 8)) // "Hello, 世"
```

**Reverse a string by runes (preserving multi-byte sequences):**

```go
func reverseByRunes(s string) string {
    runes := []rune(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    return string(runes)
}
fmt.Println(reverseByRunes("Hello, 世界!")) // "!界世 ,olleH"
```

**Case-insensitive equality with Unicode awareness:**

```go
import "strings"

if strings.EqualFold(s1, s2) {
    // equal under Unicode case folding
}
```

`EqualFold` handles the complexities of case folding for many scripts. For more advanced normalization-sensitive comparison:

```go
import "golang.org/x/text/unicode/norm"

normalized1 := norm.NFC.String(s1)
normalized2 := norm.NFC.String(s2)
if normalized1 == normalized2 {
    // canonical equivalent
}
```

**Efficiently building a large CSV row with `strings.Builder`:**

```go
func buildRow(fields []string) string {
    var b strings.Builder
    b.Grow(256) // estimated size
    for i, f := range fields {
        if i > 0 {
            b.WriteByte(',')
        }
        b.WriteString(f)
    }
    return b.String()
}
```

### 9. Summary & Exercises

Go strings are immutable byte sequences conventionally holding UTF-8. The fundamental units are the **byte** (`uint8`) and the **rune** (`int32`), with no separate character type. This model puts encoding in your hands, making costs explicit and eliminating hidden Unicode processing. Key takeaways:

- `len()` returns bytes; `utf8.RuneCountInString` returns code points.
- `for range` iterates runes efficiently.
- Slicing operates on bytes; mind rune boundaries.
- Conversions between string and `[]byte`/`[]rune` copy data.
- Use `strings.Builder` for incremental construction, normalize text for reliable comparison, and validate UTF-8 when integrity matters.

**Exercises:**

1. **Unicode-aware word counter.**
   Write a function `WordCount(s string) int` that counts the number of words in a Unicode string. Use `unicode.IsSpace` to detect whitespace boundaries, correctly handling non-ASCII spaces (e.g., non-breaking space U+00A0, ideographic space U+3000). Your function should produce the correct count even if the string is not NFC-normalized.

2. **Thread-safe string interning pool.**
   Build a type `InternPool` with methods `Get(s string) string` and `Stats() (count int, savedBytes uint64)`. `Get` returns a pointer-identical string for equal content, deduplicating memory. Use a `sync.RWMutex` to protect a `map[string]string`. The `Get` method should hold a read lock for lookup, then a write lock only when inserting a new string. Measure the memory savings for a workload of 100,000 repeated log messages.

3. **Rune-safe split by max chunk size.**
   Implement `SplitByRuneChunks(s string, chunkSize int) []string` that splits `s` into chunks of at most `chunkSize` runes, ensuring no chunk breaks a multi-byte rune. If `chunkSize <= 0`, return `nil`. The function should not allocate intermediate `[]rune` slices for the entire string; use iteration and a `strings.Builder`. Write a table-driven test that verifies the chunks, and include edge cases like empty string, chunk size larger than the number of runes, and strings with invalid UTF-8 (where the chunk should advance one byte for the invalid sequence).
