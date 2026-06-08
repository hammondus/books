## Chapter 9. Maps

Maps are Go’s built-in associative data structure, mapping keys of a comparable type to values of any type. They are as fundamental to Go programs as slices, and they carry a philosophy of “just use the built-in” that removes decisions about hash table libraries. But behind the simple `map[K]V` syntax lies a carefully tuned runtime implementation, nuanced iteration semantics, and a handful of subtle pitfalls that every seasoned engineer should understand.

This chapter dissects maps from creation to concurrency, from hashing to “map leaks,” so that you can deploy them confidently in production systems.

---

### 1. Basic Usage

A map variable has zero value `nil`; reading from a nil map returns the zero value of the value type, but writing to a nil map causes a runtime panic. Therefore you almost always create a map with `make` or a composite literal.

```go
// Creation and initialization
var nilMap map[string]int           // nil, reads OK, writes panic
m1 := make(map[string]int)          // empty, ready to use
m2 := make(map[string]int, 128)     // pre-allocate space for ~128 entries
m3 := map[string]int{
    "alpha": 1,
    "beta":  2,
}
```

**Insert and update** use the familiar indexing syntax:

```go
m1["gamma"] = 3      // insert
m1["alpha"] = 10     // update
```

**Lookup** returns the value and a boolean that indicates whether the key existed:

```go
v, ok := m1["alpha"]
if !ok {
    fmt.Println("alpha not found")
}
// Or just the value (zero if absent):
v = m1["alpha"]
```

**Deletion** uses the built-in `delete`; it’s a no‑op if the key is absent:

```go
delete(m1, "gamma")
```

**Length** is `len(m1)`, O(1). **Iteration** uses `range`:

```go
for k, v := range m1 {
    fmt.Println(k, v)
}
```

Go 1.21 introduced the `clear` built‑in for maps, which deletes all entries but retains the underlying storage (the map remains non‑nil and ready for reuse):

```go
clear(m1) // removes all key/value pairs
```

Because the map remains allocated, `clear` is cheap; if you want to release the memory back to the GC, simply set the map variable to a new empty map or `nil`.

---

### 2. Under the Hood

Maps are implemented as hash tables. The runtime representation lives in `runtime/map.go` and centres on the `hmap` struct. Understanding the machinery helps you reason about growth, iteration order, and performance.

**The hmap**
An `hmap` holds the number of elements (`count`), flags, the hash seed, the log₂ of the number of buckets (`B`), and a pointer to an array of buckets. Each bucket (`bmap`) stores up to 8 key/value pairs, plus a `tophash` array (one byte per slot) that caches the high byte of each key’s hash, allowing the runtime to quickly reject non‑matching keys without comparing full keys.

**Lookup process**
1. The runtime hashes the key using a randomised seed (chosen at map creation) to harden against hash‑flooding attacks.
2. The low bits of the hash select a bucket; the high byte (the `tophash`) is compared against the cached tophash bytes in that bucket.
3. If the tophash matches, the full key is compared (using the type’s equality operator).
4. If the bucket overflows (more than 8 entries), a chain of overflow buckets is traversed.

**Insertion and growth**
When the average number of entries per bucket (the load factor) exceeds 6.5, or when there are too many overflow buckets, the map *grows* by doubling the number of buckets. To avoid large latency pauses, growth happens incrementally: the bucket array is allocated, and old buckets are *evacuated* lazily during subsequent `mapassign` and `mapdelete` operations. This “incremental rehashing” means that a map never blocks for a full table rebuild.

**Deletion and shrinking**
Notably, maps **never shrink** automatically. Deleting entries merely marks the slot empty and may allow the bucket to be reused, but the bucket array remains the same size. This is the root cause of the “map leak” problem.

**Iteration order**
Iteration walks the bucket array sequentially. However, the runtime intentionally introduces randomness: the starting bucket and the offset within each bucket are randomised on each `range` invocation. This prevents programmers from depending on iteration order, which would silently break if the map resized or if the hash seed changed. It also mitigates certain classes of attacks that exploit predictable ordering.

---

### 3. Why This Design?

**Built‑in, not a library**
Go’s decision to make maps a first‑class language feature (with syntax and built‑ins) stems from the same “less is more” philosophy that gave us slices. When the language provides maps, every Go programmer uses the same abstraction; there’s no decision fatigue between `HashMap`, `TreeMap`, `LinkedHashMap`. The compiler and runtime can optimise map operations aggressively because they know the exact semantics.

**No parameterisation (pre‑generics)**
Before Go 1.18, maps were the only generic container. Their special‑case syntax (`map[K]V`) was a pragmatic compromise: it solved the most pressing need for a generic collection without adding a full generics system. Even today, with generics available, the built‑in map remains simpler and more performant for most use cases.

**Randomised iteration**
Random iteration order is a defensive design choice. In languages where iteration order is deterministic (Python before 3.7, or insertion‑ordered maps in Java), developers often write code that accidentally relies on that order. Randomising it at runtime forces you to write order‑independent code from the start. It also raises the bar for attacks that rely on predictable hash table layout.

**No automatic shrink**
The runtime intentionally avoids shrinking maps because shrink decisions are highly workload‑dependent and would add cost to every deletion. Instead, the language gives you tools: `clear` (1.21+) to empty the map while retaining storage, and reassignment to a new map to release memory. The philosophy: make the common path fast, and give the programmer control over expensive operations.

**No concurrent access by default**
Maps are not safe for concurrent goroutines. The runtime detects concurrent writes and panics with “concurrent map writes” to surface the bug early. Rather than pay the cost of fine‑grained locking on every operation (which is rarely needed), Go expects you to synchronise externally. The `sync` package provides `sync.Map` for specific concurrent patterns.

**Comparable keys only**
Keys must be comparable with `==` and `!=`. This rules out slices, maps, and functions as keys. The restriction is necessary because the hash table must be able to distinguish keys by equality; supporting deep comparison of slices would be expensive and semantically messy. If you need a slice as a key, convert it to a string (e.g., using `string([]byte{…})` or a custom hash).

---

### 4. Competing Approaches

| Language   | Primary map type           | Concurrency story                          | Growth/shrink behaviour                    | Iteration order                     |
|------------|----------------------------|--------------------------------------------|--------------------------------------------|-------------------------------------|
| **Java**   | `HashMap`, `ConcurrentHashMap` | `HashMap` – not thread‑safe; `ConcurrentHashMap` with striped locking | Rehashes on load factor threshold; does not shrink automatically unless you re‑create | Undefined (but typically bucket order) |
| **Python** | `dict`                     | GIL makes single‑threaded access safe, but no thread‑safe guarantees for multi‑step operations | Dynamically resizes; since 3.7 insertion order is guaranteed | Insertion order (since 3.7)        |
| **C++**    | `std::unordered_map`       | Not thread‑safe; must use external locking   | Bucket growth only; no automatic shrink; `rehash` control available | Unspecified (typically bucket order) |
| **Rust**   | `std::collections::HashMap`| Not thread‑safe; `RwLock<HashMap>` or `dashmap` crate for concurrent | Uses a Swiss table (fast, compact); grows but does not shrink | Arbitrary, based on hash seed    |
| **Go**     | `map[K]V`                  | Runtime panics on concurrent writes; `sync.Map` for special cases | Incremental growth; never shrinks automatically | Deliberately randomised         |

**Key differences:**

- **Java** separates concerns with a rich hierarchy of `Map` implementations; Go provides exactly one. This forces consistency but removes flexibility.
- **Python’s** insertion‑ordered dicts are convenient for caching and serialisation; Go’s randomised order pushes you toward explicit ordering (e.g., sorting keys) when determinism matters.
- **C++** gives you low‑level control (hash function, bucket count, max load factor), but the interface is complex. Go hides these details behind a simple API, optimising for the 95% case.
- **Rust** shares Go’s philosophy of not providing a concurrent map in the standard library, but its ownership model prevents data races at compile time. Go detects them at runtime and panics, which is a deliberate trade‑off for developer velocity.
- **Go’s `sync.Map`** fills a niche: when entries are written once but read many times, or when multiple goroutines read/write largely disjoint key sets, `sync.Map` avoids the overhead of a full `sync.RWMutex`.

---

### 5. Common Mistakes

**1. Writing to a nil map**
```go
var m map[string]int
m["key"] = 1  // panic: assignment to entry in nil map
```
Always initialise maps with `make` or `{}`.

**2. Assuming iteration order**
```go
m := map[string]int{"a":1, "b":2}
for k := range m {
    fmt.Print(k) // order is unpredictable
}
```
If you need sorted output, extract the keys, sort them, then iterate.

**3. Taking the address of a map element**
```go
m := map[string]int{"a": 1}
_ = &m["a"] // compile error: cannot take address of m["a"]
```
Map entries may be relocated during growth, so the address would become invalid. You can store pointers as values if you need indirection.

**4. Using non‑comparable types as keys**
```go
type person struct {
    name string
    tags []string // slice is not comparable
}
m := map[person]bool{} // compile error
```
If you must use a struct with a slice, consider extracting a comparable canonical key (e.g., a string or integer).

**5. Inserting during range (and expecting deterministic behaviour)**
```go
m := map[int]bool{1: true}
for k := range m {
    if k == 1 {
        m[2] = true // 2 may or may not appear in this loop
    }
}
```
The language spec says inserted entries *may* be produced during iteration; never rely on a particular outcome.

**6. Map memory “leak”**
```go
m := make(map[int][1024]byte)
for i := 0; i < 1_000_000; i++ {
    m[i] = [1024]byte{}
}
for i := 0; i < 1_000_000; i++ {
    delete(m, i) // map still occupies ~1 GB of bucket memory
}
```
Deleting all entries does not shrink the bucket array. The map retains the memory. To release it, assign `m = nil` or `m = make(map[int][1024]byte)`.

**7. Unprotected concurrent access**
```go
var m = make(map[string]int)
go func() { m["a"] = 1 }()
go func() { m["b"] = 2 }()
```
This almost certainly triggers a `fatal error: concurrent map writes`. Use a mutex or `sync.Map`.

---

### 6. Performance Considerations

**Time complexity**
- Lookup, insertion, and deletion: **amortised O(1)**. In the degenerate case of many hash collisions leading to long overflow chains, worst‑case can degrade toward O(n), but the randomised hash seed makes such collisions extremely unlikely unless an attacker deliberately crafts keys. In practice, map operations are fast and predictable.

**Allocation and growth**
- A new `map[K]V` allocates a small `hmap` header on the heap; buckets are heap‑allocated as needed.
- Every growth doubles the bucket count and involves copying existing entries to new buckets. This work is spread across future operations, but each operation that triggers an evacuation pays a small extra cost.
- **Pre‑allocation** via `make(map[K]V, hint)` reduces the number of growth steps if you know the final size, lowering total allocations and CPU. Even a rough estimate is better than none.

**Key hashing cost**
- Small, fixed‑size keys (integers, pointers, small strings) hash quickly. Large structs used as keys force the hashing function to read the entire struct, which can be expensive and cache‑unfriendly. Prefer pointer keys or a lightweight hash derived from a subset of fields if performance matters.
- String keys are hashed from the string’s bytes; the runtime caches the hash of a string once computed, so repeated lookups with the same string instance are fast.

**Cache locality**
- A bucket stores keys and values interleaved (key1, key2, …, val1, val2, …), which improves cache locality when both are accessed. When values are large, consider storing pointers instead of values to keep the bucket compact and reduce copying during growth.

**Iteration performance**
- Iteration is linear in the number of buckets plus overflow chains, not just in the number of elements. A map that has grown and then been partially cleared may still have many empty buckets; iterating it still walks the full bucket array. Regularly clearing a large map with `clear` doesn’t shrink it, so if you iterate often, you may see stale performance. To reclaim those buckets, replace the map with a new one.

**Map leak (memory bloat)**
- As discussed, maps only grow. A long‑lived map that sees bursts of high cardinality followed by mass deletions will consume memory indefinitely. Use a fresh map or periodically “copy and swap” (copy live entries to a new map and replace the original) if you cannot tolerate the footprint.

**Concurrent reads**
- Under `sync.RWMutex`, concurrent reads scale well. For read‑heavy, write‑rare workloads, `sync.Map` can outperform an `RWMutex`‑protected map because it uses a read‑only copy and atomic operations to avoid lock contention during reads.

---

### 7. Best Practices

**Initialisation**
- Use `make(map[K]V)` or a literal. Never rely on a nil map for writes.
- Provide a capacity hint when the final size is known: `make(map[string]int, 1000)`.

**Existence check**
- Always use the two‑value assignment to distinguish a missing key from a zero value:
  ```go
  if v, ok := m[key]; ok {
      // ...
  }
  ```

**Sets**
- Use `map[K]struct{}` as a set. `struct{}` takes zero bytes, making it the most memory‑efficient way.
  ```go
  set := make(map[string]struct{})
  set["a"] = struct{}{}
  if _, exists := set["a"]; exists { /* ... */ }
  ```

**Clearing**
- In Go 1.21+, use `clear(m)` to remove all entries while keeping the map’s memory. If you need to free memory, assign a new map or `nil`.

**Concurrency safety**
- For low‑contention maps, a `sync.RWMutex` is the idiomatic choice: `mu.RLock()` for reads, `mu.Lock()` for writes.
- For write‑once, read‑many patterns, or maps where goroutines touch disjoint key sets, consider `sync.Map`. It exposes `Load`, `Store`, `LoadOrStore`, `Delete`, and `Range`.
- When building high‑throughput concurrent maps, shard a regular map behind an array of mutexes to reduce contention.

**Iteration determinism**
- When order matters, extract keys into a slice, sort it, then iterate:
  ```go
  keys := make([]string, 0, len(m))
  for k := range m {
      keys = append(keys, k)
  }
  sort.Strings(keys)
  for _, k := range keys {
      _ = m[k]
  }
  ```

**Avoid large value copies during range**
- Range copies each value. If the value is a large struct, iterate with the index only and access the value via `m[key]`, or store pointers as values.

**Clean up maps that can grow unbounded**
- Implement TTL‑based eviction or periodic recreation for long‑running caches to avoid silent memory bloat.

---

### 8. Examples

**Word frequency counter** (classic map usage, with existence check)
```go
func wordCount(words []string) map[string]int {
    freq := make(map[string]int, len(words))
    for _, w := range words {
        freq[w]++
    }
    return freq
}
```

**Set operations using `map[K]struct{}`**
```go
type Set map[string]struct{}

func NewSet(keys ...string) Set {
    s := make(Set, len(keys))
    for _, k := range keys {
        s[k] = struct{}{}
    }
    return s
}

func (s Set) Add(key string) { s[key] = struct{}{} }
func (s Set) Contains(key string) bool {
    _, ok := s[key]
    return ok
}
```

**Concurrent map with `sync.RWMutex`**
```go
type SafeCache struct {
    mu sync.RWMutex
    m  map[string]any
}

func (c *SafeCache) Get(key string) (any, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    v, ok := c.m[key]
    return v, ok
}

func (c *SafeCache) Set(key string, value any) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.m[key] = value
}
```

**Preventing map memory bloat by copying live entries**
```go
func compactMap(original map[int]bool) map[int]bool {
    newMap := make(map[int]bool, len(original))
    for k, v := range original {
        newMap[k] = v
    }
    return newMap // original becomes eligible for GC if no other references
}
```

---

### 9. Summary & Exercises

**Summary**
- Maps are built‑in hash tables with `O(1)` average‑case operations.
- They never shrink automatically; memory can accumulate if you churn keys.
- Iteration order is deliberately randomised; sort keys when determinism is needed.
- Maps are not safe for concurrent access; synchronise with a mutex or use `sync.Map` for specialised workloads.
- Always initialise maps before writing, use the comma‑ok pattern for lookups, and prefer `map[K]struct{}` for sets.

**Exercises**

1. **Sharded concurrent map**
   Implement a generic concurrent map `ConcurrentMap[K comparable, V any]` that uses a fixed number of shards (e.g., 32), each protected by a `sync.RWMutex`. The map should choose the shard based on the hash of the key. Provide `Get`, `Set`, `Delete`, and `Range` (which iterates over all entries while holding a read lock per shard sequentially). Benchmark it against a single‑mutex map under varying read/write ratios and key distributions. What trade‑offs emerge?

2. **Build a TTL‑based cache with leak prevention**
   Design a cache that stores values with an expiry time. Use a background goroutine or lazy eviction to remove expired entries. Crucially, implement a mechanism that shrinks the underlying map when too many stale entries have accumulated—either by periodically replacing the map or by implementing a copy‑compact strategy. Measure memory usage over time under high churn and compare with a naive map‑only cache.

3. **Set algebra**
   Create a set type `Set[T comparable]` backed by a map, with methods `Union`, `Intersection`, `Difference`. Ensure the implementation does not modify the receiver (immutable style) and is efficient for large sets. Then make it safe for concurrent reads without locks by using copy‑on‑write semantics: updates create a new map and atomically swap the set’s backing map pointer. Discuss the scenarios where this approach outperforms a `RWMutex`.
