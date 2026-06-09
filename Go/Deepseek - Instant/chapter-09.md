## Chapter 9: Maps

Maps are Go’s built-in associative data structure, mapping keys to values. Unlike slices (where you index by contiguous integers), maps allow arbitrary key types—strings, integers, custom comparable types—to index values with amortised constant-time lookups. For the seasoned engineer, maps feel familiar (akin to dictionaries or hash tables elsewhere), but Go’s implementation carries specific trade-offs, performance characteristics, and surprising behaviours that demand respect.

### 1. Basic Usage

Maps are reference types. You declare them with `map[KeyType]ValueType`, initialise them via a composite literal or `make`, and then operate with direct syntax.

```go
package main

import (
    "fmt"
    "log"
    "slog"
)

func main() {
    // Declaration (nil map)
    var scores map[string]int
    // Reading from a nil map returns the zero value (safe)
    fmt.Println(scores["alice"]) // 0

    // But writing panics:
    // scores["alice"] = 10 // panic: assignment to entry in nil map

    // Initialisation using make
    scores = make(map[string]int, 8) // pre-allocate capacity hint
    scores["alice"] = 10
    scores["bob"] = 20

    // Composite literal
    ages := map[string]int{
        "alice": 30,
        "bob":   25,
    }

    // Lookup with comma-ok idiom (distinguishes missing key from zero value)
    if age, ok := ages["alice"]; ok {
        fmt.Printf("Alice is %d years old\n", age)
    } else {
        fmt.Println("Alice not found")
    }

    // Delete an entry
    delete(ages, "bob")

    // Iteration order is random
    for key, val := range ages {
        fmt.Printf("%s -> %d\n", key, val)
    }

    // Using maps with structured logging (slog)
    slog.Info("user stats", "scores", scores)
}
```

**Key syntax points:**
- `make(map[K]V, hint)` – hint is initial capacity; avoids early growth.
- `v, ok := m[k]` – `ok` is `true` if the key exists.
- `delete(m, k)` – removes key (no-op if absent).
- `len(m)` – number of keys.
- Maps are **not safe for concurrent writes**. Use `sync.RWMutex` or `sync.Map`.

### 2. Under the Hood

A Go map is a pointer to a runtime structure defined in `runtime/map.go`: the `hmap` struct. Understanding this changes how you write high-performance code.

**The `hmap` structure (simplified):**
```go
type hmap struct {
    count     int    // number of live entries
    flags     uint8
    B         uint8  // log_2 of number of buckets (buckets = 2^B)
    noverflow uint16 // approximate number of overflow buckets
    hash0     uint32 // hash seed (random per map, mitigates hash flooding)

    buckets    unsafe.Pointer // array of 2^B bucket structs
    oldbuckets unsafe.Pointer // previous bucket array (during growth)
    nevacuate  uintptr        // progress counter for growth

    extra *mapextra // optional overflow bucket data
}
```

Each **bucket** holds up to 8 key-value pairs. When a bucket overflows (9th entry), Go allocates an overflow bucket and chains it.

**Hash function:** For each key, Go computes a hash using the runtime’s `memhash` (per-map `hash0` seed). The lower bits (`B` bits) select the bucket. Within a bucket, the top 8 bits of the hash (`tophash`) are stored in an array, enabling fast lookup without comparing full keys.

**Lookup process:**
1. Compute hash of key.
2. Use low bits to pick bucket.
3. Within bucket, compare `tophash` values; if match, compare full key (pointer equality then deep equality for strings etc.).
4. If not found, follow overflow chain.
5. If the map is growing (`oldbuckets != nil`), also check the old bucket (may be evacuated).

**Growth strategies:**
- **Load factor > 6.5** triggers bucket doubling (incremental, amortised). Each write may evacuate one or two buckets.
- **Too many overflow buckets** (when many deletions create sparse buckets) triggers `same-size growth` – reorganises into new buckets to reduce overflow chains.

**Memory layout nuance:** Maps never shrink. Even after `delete` removes most keys, the bucket array stays at its grown size. This is the source of “map leak” – a map that transiently held millions of keys will retain that memory indefinitely.

**Key equality:** Keys must be **comparable** (== operator defined). All basic types, pointers, channels, arrays of comparable types, and structs composed of comparable fields are valid. Slices, maps, and functions are not comparable → cannot be map keys (except via `reflect` or `unsafe`, which you should almost never do).

### 3. Why This Design?

Go’s map design reflects the language’s philosophy: **built-in, safe, and predictable with explicit trade-offs**.

- **Why built-in, not a library type?**  
  Maps are so fundamental that having `map[T]U` as syntax (rather than `HashMap[T,U]`) reduces friction. Generic types arrived in Go 1.18, but maps were part of the language since 1.0 – the team prioritised ergonomics over “purity”. The compiler can also optimise map operations (e.g., inlining the hash lookup).

- **Why no `shrink` operation?**  
  Go deliberately avoids automatic shrinking to prevent “thrashing” – a map that repeatedly grows and shrinks would cause CPU and GC churn. The team expects the programmer to reason about lifecycle: if a map is temporary, let it be GC’d; if long-lived with churn, consider copying to a fresh map or using a different data structure.

- **Why random iteration order?**  
  Early Go versions had deterministic iteration (starting from the first bucket). Programmers unintentionally relied on that order, leading to subtle bugs when map implementation changed. By randomising the start bucket and iteration direction (since Go 1.12, the randomisation is per‑iteration), the Go team forces you to never depend on order. This aligns with “make bugs obvious.”

- **Why no concurrency safety?**  
  Adding mutexes inside every map operation would slow down single‑goroutine use (the common case) and still not solve the need for transactional updates. Go’s answer: “share memory by communicating” – use channels to serialise access, or add your own `sync.RWMutex`. The race detector catches accidental concurrent access.

### 4. Competing Approaches

| Language | Map Type | Key Restrictions | Concurrency | Growth Behaviour |
|----------|----------|------------------|-------------|------------------|
| **Go** | `map[K]V` | Comparable types | Not safe; use `sync.Map` or mutex | Grows only, never shrinks |
| **Python** | `dict` | Hashable (immutable by value) | GIL protects dict methods (but not atomicity) | Resizes; can shrink with `dict.copy()` or `dict.clear()` |
| **Java** | `HashMap<K,V>` | `equals` & `hashCode` | Not safe; `ConcurrentHashMap` for concurrent | Grows; never shrinks (but can rehash) |
| **Rust** | `std::collections::HashMap` | `Hash + Eq` | Not safe; `RwLock<HashMap>` or `DashMap` | Grows; `shrink_to_fit` available |
| **C++** | `std::unordered_map` | `std::hash` + `operator==` | Not safe; external locking | Rehashes on load factor; no shrink |

**Key differentiators:**
- **Python** prioritises convenience – any hashable object works (including custom classes). The GIL serialises bytecode, so concurrent writes can corrupt internal state (but CPython 3.12+ has per‑dict locking in some cases).  
- **Java’s `ConcurrentHashMap`** uses fine‑grained locking (striping) – Go deliberately avoids this complexity in the stdlib, offering `sync.Map` as an optimised but special‑case alternative.  
- **Rust** gives you control: you can call `shrink_to_fit` explicitly. Go philosophically avoids “options” – you either accept the growth or design around it.

**What Go does differently:** The `comma-ok` idiom (`v, ok := m[k]`) is baked into the language, not a method call. This makes zero-value handling explicit and avoids “key not found” exceptions.

### 5. Common Mistakes

**5.1. The Map Leak (Memory Never Shrinks)**  
A map that grew to 1 million entries, then `delete`’d all but 10, still holds the large bucket array. This memory is neither returned to the OS nor reused for other maps.  
*Fix:* Periodically recreate the map: `fresh := make(map[K]V); for k, v := range oldMap { fresh[k] = v }` and replace the reference.

**5.2. Assuming Iteration Order**  
Even though iteration order is random, it’s not truly random across different runs – Go uses a per‑iteration seed. But it’s still non‑deterministic. Never write tests that depend on order.

**5.3. Concurrent Read/Write Panic**  
Reading while a write happens (or vice versa) causes a fatal panic, not a race. The detection is eager: Go’s runtime checks concurrent access to the same map and panics immediately.  
*Fix:* Use `sync.RWMutex` or `sync.Map`. For high‑contention scenarios, consider sharding (multiple maps keyed by a hash prefix).

```go
var mu sync.RWMutex
m := make(map[string]int)

// Write
mu.Lock()
m["key"] = 42
mu.Unlock()

// Read
mu.RLock()
val := m["key"]
mu.RUnlock()
```

**5.4. Using a Map as a Key with Pointer Fields**  
Two different `struct` values with pointer fields that point to equal content are **not equal** because pointers are compared by address, not by dereferenced value. This leads to missing keys.  
*Fix:* Use a custom key type that implements deterministic equality, or store a string representation.

**5.5. Taking the Address of a Map Element**  
`&m["key"]` does not compile – map elements are not addressable (because growth moves them). If you need a pointer, store pointers in the map: `map[string]*T`.

**5.6. `nil` vs Empty Map**  
Reading from a `nil` map is fine (returns zero value), but writing panics. Often you want to initialise with `make` even for empty maps that will receive writes.

### 6. Performance Considerations

**Time complexity:**  
- Average case: **O(1)** per lookup/insert/delete.  
- Worst case (pathological hash collisions): O(n) – mitigated by random seed and per‑map hash salt.

**Memory overhead:**  
- Each entry costs ~72 bytes on 64‑bit systems (key pointer, value pointer, overhead).  
- Empty map still has a pointer (8 bytes) to an `hmap`.  
- Overflow buckets add extra allocations – high load factor degrades performance.

**Allocation patterns:**
- `make(map[K]V, hint)` pre‑allocates buckets, avoiding early rehashes. Hint should be a reasonable upper bound.
- Maps of **pointers** to large structs reduce copy cost and GC scanning (the map stores the pointer, not the struct).
- Maps of **interfaces** (`map[string]interface{}`) incur boxing and type‑assertion costs.

**Comparison of key types (speed, fastest to slowest):**  
1. **Integer** – fastest hash, direct equality.  
2. **String** – hashing requires O(len) but cached in string header.  
3. **Struct with comparable fields** – hash combines field hashes.  
4. **Array** – slowest (hashes each element).

**GC impact:**  
- The map’s internal bucket array is a single large allocation (if pre‑allocated).  
- Overflow buckets become many small allocations – increase GC pressure.  
- Keys and values that are pointers keep those objects live.

**Practical optimisation:**
- Prefer `map[string]T` over `map[MyStruct]T` when MyStruct has many fields.
- If you frequently need to take the union or difference of maps, consider using two‑slice representations for iteration.
- For maps that grow and then become read‑only, switch to a custom read‑only lookup (e.g., sorted slice + binary search) after freezing.

### 7. Best Practices (Idiomatic Go)

**7.1. Always use the comma‑ok idiom** when distinguishing zero values from missing keys.

```go
// Bad: confuses missing key with value 0
score := scores["alice"]

// Good
score, ok := scores["alice"]
if !ok {
    // handle missing
}
```

**7.2. Preallocate capacity** when you know the approximate size.

```go
// Prevents multiple rehashes for 1000 entries
m := make(map[string]int, 1000)
```

**7.3. Use `clear()` built‑in (Go 1.21+)** to delete all entries while retaining the same bucket array (avoids reallocation but doesn’t shrink).

```go
clear(m)  // equivalent to for k := range m { delete(m, k) }
```

**7.4. Avoid storing large values directly – store pointers instead.**  
Copying a 1KB struct into the map on each insert is expensive. Store `*LargeStruct`.

**7.5. Use `sync.Map` only for specific patterns:**  
- Write-once, read-many (e.g., cached config).  
- Multiple goroutines reading and writing disjoint key sets.  
For all other cases, a regular map with `sync.RWMutex` is simpler and often faster.

**7.6. Don’t return maps from APIs that the caller should not modify.**  
Return a copy or an immutable view (e.g., a function that looks up values). If you must return a map, document that modifications affect the internal state.

**7.7. Use `map[K]struct{}` to implement a set** – zero memory overhead for the value.

```go
set := make(map[string]struct{})
set["alice"] = struct{}{}
if _, ok := set["alice"]; ok {
    fmt.Println("alice exists")
}
```

### 8. Examples

**Example 1: Thread‑safe cache with TTL (using mutex and map)**

```go
type Cache struct {
    mu    sync.RWMutex
    items map[string]cacheItem
    ttl   time.Duration
}

type cacheItem struct {
    value      interface{}
    expiresAt  time.Time
}

func NewCache(ttl time.Duration) *Cache {
    return &Cache{
        items: make(map[string]cacheItem),
        ttl:   ttl,
    }
}

func (c *Cache) Set(key string, value interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = cacheItem{
        value:     value,
        expiresAt: time.Now().Add(c.ttl),
    }
}

func (c *Cache) Get(key string) (interface{}, bool) {
    c.mu.RLock()
    item, ok := c.items[key]
    c.mu.RUnlock()
    if !ok || time.Now().After(item.expiresAt) {
        // Delete expired (could do in background goroutine)
        c.mu.Lock()
        delete(c.items, key)
        c.mu.Unlock()
        return nil, false
    }
    return item.value, true
}
```

**Example 2: Efficiently merging two maps without extra allocations**

```go
func mergeMaps(dst, src map[string]int) {
    // Assumes dst is pre-sized large enough
    for k, v := range src {
        dst[k] = v
    }
}
```

**Example 3: Using a map to deduplicate a slice (order not preserved)**

```go
func uniqueStrings(in []string) []string {
    seen := make(map[string]struct{}, len(in))
    out := make([]string, 0, len(in))
    for _, s := range in {
        if _, ok := seen[s]; !ok {
            seen[s] = struct{}{}
            out = append(out, s)
        }
    }
    return out
}
```

### 9. Summary & Exercises

**Summary:**  
Maps in Go are fast, built‑in hash tables with a “value semantics” syntax but reference‑type behaviour. They grow only, never shrink, are not concurrency‑safe, and have random iteration order. Mastering the comma‑ok idiom, preallocation, and understanding the `hmap` structure will prevent performance pitfalls and memory leaks.

**Exercises:**

1. **Implement a sharded map** to reduce lock contention. Create a `ShardedMap` that splits keys into N shards (e.g., 256), each with its own `sync.RWMutex` and `map[string]interface{}`. Provide `Get` and `Set` methods that hash the key to choose a shard. Benchmark against a single mutex‑protected map with `go test -bench=. -race`.

2. **Write a map that automatically evicts the least recently used (LRU) entry** when it exceeds a given capacity. Use a combination of a map (for O(1) lookup) and a doubly linked list (for LRU order). The standard library’s `container/list` can help. Ensure all operations are O(1). Test with a concurrent workload.

3. **Debug a memory leak:** Given a service that processes 1 million unique user IDs per hour via a map that accumulates IDs (and never clears them), simulate this leak in a short program. Then fix it using two different strategies:  
   - Periodically re‑create the map.  
   - Use a `map[string]struct{}` with a separate time‑based sweep goroutine.  
   Measure memory usage with `runtime.MemStats` and `pprof`.
