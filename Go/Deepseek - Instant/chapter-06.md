## Chapter 6: Control Flow

Control flow in Go deliberately breaks from the traditions of C-family languages. While Go provides the familiar `if`, `for`, and `switch`, it strips away redundancy and introduces subtle semantics that affect correctness, performance, and readability. This chapter covers how to wield these constructs idiomatically, what happens under the compiler’s hood, and why Go’s designers chose to include exactly one looping keyword.

### 1. Basic Usage

Go’s control flow statements look familiar but behave differently in key ways.

#### `if` Statements

`if` statements can include an initialization clause (scoped to the `if`/`else` chain) and never require parentheses around the condition. Braces are mandatory.

```go
// Basic form
if x > 10 {
    return "large"
}

// With initialization (scoped to if/else)
if err := process(); err != nil {
    return fmt.Errorf("process failed: %w", err)
} else {
    // err is still accessible here
    log.Printf("process succeeded with %v", err)
}

// No ternary operator; use an if-else expression
result := "default"
if condition {
    result = "truthy"
}
```

#### `switch` Statements

Go’s `switch` does **not** fall through to the next case by default. Each case is a separate block. You can use `fallthrough` explicitly when needed.

```go
// Expression switch
switch day {
case "Monday":
    return 1
case "Tuesday", "Wednesday": // Multiple values
    return 2
default:
    return 0
}

// Expressionless switch (acts as if-else chain)
switch {
case score >= 90:
    grade = "A"
case score >= 80:
    grade = "B"
default:
    grade = "F"
}

// Type switch
var v any = "hello"
switch t := v.(type) {
case string:
    fmt.Printf("string of length %d", len(t))
case int:
    fmt.Printf("int: %d", t)
default:
    fmt.Printf("unknown type %T", t)
}
```

#### `for` – The Only Looping Construct

Go has no `while` or `do-while`. `for` serves all iteration needs.

```go
// Complete for (C-style)
for i := 0; i < 10; i++ {
    fmt.Println(i)
}

// While-style (condition only)
for scanner.Scan() {
    fmt.Println(scanner.Text())
}

// Infinite loop
for {
    select {
    case <-ctx.Done():
        return
    default:
        work()
    }
}
```

#### `range` Clause

`range` works with slices, arrays, maps, strings, and channels, producing one or two iteration variables.

```go
// Slice/array: index, value copy
for idx, val := range slice {
    // val is a copy
}

// Map: key, value copy (order non-deterministic)
for key, val := range myMap {
    // iteration order is randomized
}

// String: index, rune (not byte)
for pos, ch := return "世界" {
    fmt.Printf("%d: %c (U+%04X)\n", pos, ch, ch)
}

// Channel: receives until channel is closed
for msg := range ch {
    process(msg)
}
```

#### `goto` and Labels

`goto` jumps to a label within the same function. It is rarely idiomatic but useful for breaking out of deeply nested loops where a labeled `break` is cleaner.

```go
func search(matrix [][]int, target int) bool {
    for i := 0; i < len(matrix); i++ {
        for j := 0; j < len(matrix[i]); j++ {
            if matrix[i][j] == target {
                goto found
            }
        }
    }
    return false
found:
    return true
}
```

### 2. Under the Hood

#### How `for` Compiles

The Go compiler transforms all `for` loops into a single internal representation: an **init statement**, a **condition**, and a **post statement**. For `while`-style `for condition {}`, the compiler treats the condition as the only component. The generated SSA (Static Single Assignment) form uses explicit `goto` instructions for the loop back-edge.

For `range` loops, the compiler desugars the construct:

- **Slice range**: `for i, v := range slice` becomes a loop that increments `i`, fetches `slice[i]` into a temporary variable `v`, and stops when `i == len(slice)`. The temporary copy of `v` means modifications to `v` do not affect the slice element.
- **Map range**: The compiler calls `runtime.mapiterinit` to obtain a hash iterator, then `runtime.mapiternext` on each iteration. The iteration order is intentionally randomized by using a random starting bucket and a hash seed.
- **Channel range**: `for v := range ch` is syntactic sugar for an infinite loop that receives from the channel and checks `ok`. The compiler emits a loop that calls `runtime.chanrecv` and terminates when the channel is closed.

#### Switch Implementation

When the number of integer cases is dense, the compiler builds a **jump table** (constant-time dispatch). For sparse or string cases, it emits a binary search or a linear comparison chain (O(log n) or O(n)). Expressionless switches with many boolean-like conditions are compiled as chained `if-else` blocks.

The `fallthrough` keyword simply bypasses the implicit `break` at the end of a case, continuing execution into the next case without re-evaluating its condition.

#### Label Scoping and `goto` Limitations

Labels are function-scoped. `goto` cannot:
- Jump into a block (e.g., inside an `if` or `for` from outside)
- Jump over variable declarations that would become uninitialized
- Skip the creation of a `defer` statement (the compiler disallows this)

The compiler verifies that a `goto` does not violate variable lifetime rules, ensuring that all variables are properly initialized along the path of execution.

### 3. Why This Design?

#### Only One Looping Construct (`for`)

The designers observed that `while` and `do-while` are redundant. In C, `while (cond) {}` is a `for` with missing init and post sections. `do-while` can be emulated with `for { ... if !cond { break } }`. Removing syntactic sugar reduces the language’s surface area and eliminates the “which loop should I use?” decision fatigue. Every Go developer reads and writes loops exactly one way.

#### No Ternary Operator (`cond ? a : b`)

Go rejects the ternary operator because it leads to dense, hard-to-read expressions, and encourages non-idiomatic code. The explicit `if-else` assignment, while verbose, makes the control flow unmistakable and plays nicely with Go’s design principle of **linear readability** (code should be scanned top-to-bottom without hidden branching inside expressions).

#### `switch` Without Fallthrough by Default

In C and Java, forgetting `break` is a common source of bugs. Go reverses the default: cases are independent, and `fallthrough` is an explicit admission that you want the dangerous behavior. This design choice has dramatically reduced switch-related defects in production Go code.

Expressionless `switch` (`switch { case cond1: ... }`) provides a cleaner alternative to a chain of `if-else if` when the conditions are not simple equality checks. It also allows the compiler to better reason about exhaustiveness (though Go does not enforce exhaustiveness like Rust’s `match`).

#### `range` Over Channels and Strings

`range` unifies iteration over diverse types into a single syntax. For strings, iterating over runes (not bytes) avoids common Unicode bugs. For channels, `range` automatically handles channel closure, freeing the programmer from manually checking the `ok` flag on each receive.

#### `goto` – Pragmatic Escape Hatch

While `goto` is discouraged, the Go team included it because complex error recovery and breaking out of deeply nested loops sometimes require it. Instead of inventing a new control flow feature, they kept `goto` as a low-level, well-understood tool.

### 4. Competing Approaches

| Feature | Go | C/Java | Rust | Python | JavaScript |
|---------|-----|--------|------|--------|------------|
| **Loop constructs** | `for` only | `for`, `while`, `do-while` | `loop`, `while`, `for` (range) | `for`, `while` | `for`, `while`, `do-while` |
| **Range iteration** | `range` (slice, map, string, channel) | Enhanced `for` (Java) | `for item in collection` | `for item in collection` | `for...of`, `for...in` |
| **Fallthrough default** | Off | On (C/Java) | N/A (match arms are separate) | N/A (no switch) | On (unless using `break`) |
| **Ternary operator** | No | Yes | Yes (if-else as expression) | Yes (if-else expression) | Yes |
| **Labeled break/continue** | Yes | Yes (limited) | Yes (with loop labels) | No | Yes (with label) |
| **Pattern matching** | Type switch only | No | Full `match` with destructuring | `match` (3.10+) | No (proposal) |

**Key observations**:
- **Java/C#** prioritize familiarity from C, but inherit the fallthrough hazard. Go’s safer `switch` trades familiarity for reliability.
- **Rust** replaces the classic `switch` with `match`, which is more powerful (exhaustiveness checking, destructuring) but also more complex. Go’s `switch` is intentionally limited, focusing on equality and type tests.
- **Python** lacks a `switch` statement entirely (until 3.10’s `match`), relying on `if-elif` chains or dictionaries of functions. Go’s `switch` fills this gap without Python’s dynamic dispatch overhead.
- **JavaScript** includes both `for...in` (enumerable properties, often misused on arrays) and `for...of` (iterables). Go’s single `range` avoids this confusion by being explicit about what each iteration produces (index/value for arrays, key/value for maps).

### 5. Common Mistakes

#### 1. Loop Variable Capture in Goroutines

The classic Go trap: starting goroutines inside a loop captures the same variable reference.

```go
// WRONG: All goroutines see the final value of 'i'
for i := 0; i < 5; i++ {
    go func() {
        fmt.Println(i) // closes over the loop variable (by reference)
    }()
}
time.Sleep(time.Second) // prints "5 5 5 5 5"

// CORRECT: Copy the variable
for i := 0; i < 5; i++ {
    go func(val int) {
        fmt.Println(val) // val is a copy
    }(i)
}
```

#### 2. Modifying Slice Elements in a `range` Value Copy

`range` copies the element value. Modifying the copy does nothing to the original slice.

```go
// WRONG: val is a copy
for _, val := range slice {
    val = transform(val) // slice unchanged
}

// CORRECT: Use index
for i := range slice {
    slice[i] = transform(slice[i])
}
```

#### 3. Assuming Map Iteration Order

Go deliberately randomizes map iteration order. Never rely on a specific sequence.

```go
// WRONG: Expecting insertion order
m := map[string]int{"a": 1, "b": 2, "c": 3}
for k, v := range m {
    fmt.Println(k, v) // order can be a,c,b or any permutation
}
```

#### 4. Using `break` Inside `switch` to Exit an Outer Loop

`break` inside a `switch` only exits the `switch`, not an enclosing loop. Use a labeled `break`.

```go
// WRONG: break only exits the switch
for i := 0; i < 10; i++ {
    switch {
    case i == 5:
        break // exits switch, not the loop
    }
}

// CORRECT: Label the loop
loop:
for i := 0; i < 10; i++ {
    switch {
    case i == 5:
        break loop // exits the loop
    }
}
```

#### 5. Misusing `fallthrough`

`fallthrough` bypasses the next case’s condition but does **not** re-evaluate it. This often leads to logic errors.

```go
switch x {
case 1:
    fmt.Println("one")
    fallthrough // always goes to case 2, even if x != 2
case 2:
    fmt.Println("two")
}
```

#### 6. `goto` Jumping Over Variable Declaration

```go
// ILLEGAL: jumps over declaration of 'a'
goto label
var a int = 42
label:
fmt.Println(a) // a is not in scope
```

### 6. Performance Considerations

#### `for` vs `range` Over Slices

- `for i := 0; i < n; i++` **does not copy elements**. Use this for large struct slices where copying would be expensive.
- `for _, v := range slice` **always copies `v`**. For `[]int`, the copy is cheap (8 bytes). For `[]SomeLargeStruct` (e.g., 128+ bytes), copying adds measurable overhead.
- The compiler can sometimes optimize away the copy for small types (≤ machine word), but not for large structs.

**Benchmark example** (concept):
```
BenchmarkForIndex-8      1000000000   0.85 ns/op   0 B/op   0 allocs/op
BenchmarkRangeCopy-8     500000000    1.80 ns/op   0 B/op   0 allocs/op // 2x slower for large struct
```

#### Map Iteration Overhead

Iterating over a map with `range` requires:
- A call to `runtime.mapiterinit` (allocates an iterator state on the heap if the map is large)
- `runtime.mapiternext` per iteration (hashing, bucket traversal)
- Copying key and value into loop variables (two memory copies)

For performance-critical loops, consider storing keys in a separate slice if you iterate repeatedly.

#### Switch Jump Table

- For `switch` on integers with dense cases (e.g., 0–100), the compiler generates a jump table (O(1) dispatch).
- Sparse integer cases become a binary search over case values (O(log n)).
- String switches compile into a perfect hash function (compile-time computed) or a linear scan.
- Expressionless switches with many conditions become linear `if-else` chains.

#### Range Over Channels

`for v := range ch` blocks until a value is received. The channel’s mutex is acquired on each iteration. For high-throughput channels (millions of messages/sec), the overhead is negligible compared to the receive itself. However, using `range` on a channel with a long delay between sends keeps the goroutine runnable (parked), which is efficient.

#### Avoiding Allocation in Loop Conditions

```go
// Good: len(slice) is O(1) and cheap
for i := 0; i < len(slice); i++ { }

// Bad: Calling a function that allocates
for i := 0; i < len(expensiveComputation()); i++ { } // computed each iteration
```

### 7. Best Practices (Idiomatic Go)

1. **Prefer `for range` for clarity when you need both index and value** – unless performance profiling proves copying is a problem.
2. **Use `for condition { }` for while-style loops**. Never emulate `do-while` with a `for { if !cond { break } }` unless the post-condition is truly required.
3. **Use expressionless `switch` instead of long `if-else if` chains** – it’s more readable and signals that the cases are mutually exclusive.
4. **Avoid `fallthrough` except in rare lexical scanners** (e.g., implementing a state machine where falling through is intentional and documented).
5. **Labeled `break` is preferred over `goto`** for exiting multiple nested loops. Reserve `goto` for low-level cleanup patterns that cannot be expressed with `defer` or labeled breaks.
6. **Always copy loop variables before goroutine capture** – either pass as a parameter or create a local copy inside the loop.
7. **Use `for i := range slice` to mutate elements** – not `for _, v := range slice`.
8. **Never assume map iteration order** – sort keys explicitly if needed.
9. **Use `for range` with channels** to simplify receive loops – it automatically handles closed channels.
10. **Keep `switch` case bodies short** – if a case exceeds 5–10 lines, extract it into a function.

### 8. Examples

#### Example 1: Clean Channel Processing with `range`

```go
// processWork reads jobs from a channel until it's closed.
// It uses a for-range loop, which terminates when ch is closed.
func processWork(ctx context.Context, ch <-chan Job) ([]Result, error) {
    var results []Result
    for job := range ch {
        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        default:
            res, err := job.Do()
            if err != nil {
                return nil, fmt.Errorf("job failed: %w", err)
            }
            results = append(results, res)
        }
    }
    return results, nil
}
```

#### Example 2: Expressionless Switch for Request Handling

```go
func handleRequest(r *http.Request) (int, string) {
    switch {
    case r.Method == http.MethodGet && r.URL.Path == "/health":
        return http.StatusOK, "ok"
    case r.Method == http.MethodGet && strings.HasPrefix(r.URL.Path, "/api/v1/users/"):
        return handleUserGet(r)
    case r.Method == http.MethodPost && r.URL.Path == "/api/v1/users":
        return handleUserCreate(r)
    default:
        return http.StatusNotFound, "not found"
    }
}
```

#### Example 3: Labeled Break to Exit Nested Search

```go
type Point struct{ X, Y int }

func FindFirst(matrix [][]int, target int) (Point, bool) {
    for i := 0; i < len(matrix); i++ {
        for j := 0; j < len(matrix[i]); j++ {
            if matrix[i][j] == target {
                return Point{X: i, Y: j}, true
            }
        }
    }
    return Point{}, false
}
```

#### Example 4: Efficient Struct Slice Summation (Avoiding Copy)

```go
type LargeStruct struct {
    Data [128]byte
    Value int64
}

// SumValues uses index access to avoid copying the 128-byte struct.
func SumValues(items []LargeStruct) int64 {
    var sum int64
    for i := range items {
        sum += items[i].Value
    }
    return sum
}
```

### 9. Summary & Exercises

#### Summary

- Go provides exactly one looping keyword (`for`) in three forms: complete, while-style, and infinite.
- `range` unifies iteration over slices, arrays, maps, strings, and channels, but copies values.
- `switch` defaults to non‑fallthrough, reducing a common bug category.
- `if` can include an initialization statement, scoping variables to the branch.
- `goto` exists but is rarely used; labeled `break` covers most multi‑level exit needs.
- Performance considerations include copy overhead in `range`, map iteration randomization, and jump table dispatch for dense `switch` cases.

#### Exercises

**Exercise 1: Channel Fan-Out with Early Termination**  
Write a function `MergeWithTimeout` that reads from two channels of type `<-chan int`, merges them into a single `chan int`, and stops after either:
- Both input channels are closed, or
- A `time.After` of 2 seconds expires.

Use `for range` loops and `select`. Ensure no goroutine leaks.

**Exercise 2: Matrix Search with Labeled Break**  
Given a 2D slice `[][]int` where rows are sorted ascending and columns are sorted ascending (young tableau), implement `Find` that returns `(row, col, true)` if target exists, otherwise `(0,0,false)`. Use a single `for` loop with labels to break out when the search space is exhausted (start from top-right corner). Do not use `goto`.

**Exercise 3: Optimize a Map Iteration Hot Path**  
The following code averages values in a map and is called millions of times. Identify the performance issue and rewrite it for speed.

```go
func avgValue(m map[string]float64) float64 {
    var sum float64
    for _, v := range m {
        sum += v
    }
    return sum / float64(len(m))
}
```

**Hint**: The issue is not in the loop body but in the map iteration overhead when the map size is large. Consider caching keys or maintaining a separate sum.

**Exercise 4 (Bonus – Design Discussion)**  
The Go team decided not to include a `do-while` loop. Provide a concrete example where a `do-while` would be more readable than Go’s `for { ... if !cond { break } }`. Then argue for or against adding it to Go based on the language’s simplicity pillar.
