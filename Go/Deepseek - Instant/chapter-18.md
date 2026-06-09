# Chapter 18: Testing & Benchmarking

Testing in Go isn't an afterthought—it's a first-class citizen embedded directly into the toolchain. Unlike ecosystems where testing frameworks are community-driven add-ons with fragmented DSLs, Go ships with `go test`, a built-in command that discovers, executes, and reports on tests with zero ceremony. This chapter explores the mechanics, philosophy, and advanced patterns of Go's testing arsenal, from table-driven tests to fuzzing and benchmarks.

---

## 1. Basic Usage: The "How"

The `testing` package requires only three conventions: filename suffix `_test.go`, function signature `func TestXxx(t *testing.T)`, and the `go test` command.

### Simple Test Example

```go
// math/math.go
package math

func Add(a, b int) int {
    return a + b
}

// math/math_test.go
package math

import "testing"

func TestAdd(t *testing.T) {
    got := Add(2, 3)
    want := 5
    if got != want {
        t.Errorf("Add(2,3) = %d, want %d", got, want)
    }
}
```

Run with:
```bash
go test ./...                    # all packages recursively
go test -v                       # verbose output
go test -run TestAdd             # run only tests matching regex
go test -count=1                 # disable test caching
```

### Table-Driven Tests

The idiomatic Go pattern for multiple test cases:

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name string
        a, b int
        want int
    }{
        {"positive numbers", 2, 3, 5},
        {"negative numbers", -1, -2, -3},
        {"zero", 0, 5, 5},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            if got := Add(tt.a, tt.b); got != tt.want {
                t.Errorf("Add(%d,%d) = %d, want %d", tt.a, tt.b, got, tt.want)
            }
        })
    }
}
```

### Subtests with `t.Run`

Subtests enable:
- Independent test execution (`go test -run TestAdd/positive_numbers`)
- Parallelism (`t.Parallel()` per subtest)
- Setup/teardown per case

```go
func TestDatabase(t *testing.T) {
    db := setupDB(t)
    t.Cleanup(func() { db.Close() }) // runs after all subtests

    t.Run("insert", func(t *testing.T) {
        t.Parallel() // runs in parallel with other subtests
        if err := db.Insert("key", "value"); err != nil {
            t.Fatal(err)
        }
    })

    t.Run("query", func(t *testing.T) {
        t.Parallel()
        val, err := db.Query("key")
        if err != nil {
            t.Fatal(err)
        }
        if val != "value" {
            t.Errorf("got %q, want %q", val, "value")
        }
    })
}
```

### Fuzzing (Go 1.18+)

Automatically generates inputs to find edge cases:

```go
// fuzz_test.go
func FuzzParseTimestamp(f *testing.F) {
    // Seed corpus
    f.Add("2024-01-15T10:30:00Z")
    f.Add("invalid")

    f.Fuzz(func(t *testing.T, ts string) {
        parsed, err := time.Parse(time.RFC3339, ts)
        if err != nil {
            t.Skip() // expected errors are fine
        }
        // Property: round-trip should be consistent
        if parsed.Format(time.RFC3339) != ts && err == nil {
            t.Errorf("Round-trip failed: %q -> %q", ts, parsed.Format(time.RFC3339))
        }
    })
}
```

Run with:
```bash
go test -fuzz=FuzzParseTimestamp -fuzztime=30s
```

### Benchmarks

Function signature `func BenchmarkXxx(b *testing.B)`. The framework runs your code `b.N` times until stable timing.

```go
// bench_test.go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(1, 2)
    }
}

// with allocation reporting
func BenchmarkStringConcat(b *testing.B) {
    for i := 0; i < b.N; i++ {
        s := "hello"
        s += " world"
        _ = s
    }
}
```

Run:
```bash
go test -bench=.                    # all benchmarks
go test -bench=Add$ -benchmem       # regex match, show allocs
go test -bench=. -count=5           # 5 runs for statistical significance
```

---

## 2. Under the Hood: The "Engine"

### Test Discovery and Execution

The `go test` command compiles each package (including `_test.go` files) into a temporary executable. Unlike interpreted test runners, this binary contains:
- All test and benchmark functions (discovered via symbol naming conventions)
- Package initialization code (`init()` functions)
- Coverage instrumentation (if `-cover` enabled)

The test binary runs inside a separate process with its own goroutine scheduler. The test driver communicates exit status and output.

### Caching Mechanism

`go test` caches successful results based on:
- Test binary contents (compiled code)
- Command-line arguments (flags like `-run`, `-count`)
- Environment variables explicitly listed by `testing.Testing()` or `testing.RegisterCover`
- File system state of package source files

To force a re-run without cache: `go test -count=1`. The cache lives in `$GOCACHE` (usually `~/.cache/go-build`).

### Parallel Execution

By default, tests run sequentially within a package. When `t.Parallel()` is called, the test is marked as parallelizable. The scheduler:
1. Runs all non-parallel tests first
2. Executes parallel tests with `GOMAXPROCS` concurrency (defaults to number of CPUs)

Within a parallel subtest, `t.Parallel()` blocks until the parent test yields control. This design prevents parallel subtests from interfering with parent setup/teardown.

### Benchmark Scaling

The benchmark framework uses an exponential probe to determine `b.N`. It starts with 1 iteration, measures duration, and multiplies `b.N` until runtime reaches at least 1 second (default). The formula:

```
next N = current N * (target duration / actual duration)
```

This adaptive mechanism ensures statistical significance without manual tuning.

### Coverage Instrumentation

When `-cover` is enabled, `go test` instruments the source code by inserting counters before each basic block. The coverage report (`-coverprofile=out.out`) records which counters incremented. Use `go tool cover -html=out.out` to visualize.

---

## 3. Why This Design?: The "Philosophy"

### Built-in Over Bolt-on

Most languages delegate testing to third-party libraries (JUnit, pytest, RSpec). Go's decision to embed testing into the standard library reflects a core tenet: **reduce friction to encourage discipline**. There's no "which test framework should I use?" debate. No configuration files for test runners. No JVM fork overhead. `go test` works identically across all Go projects.

### No Assertion Library

Go's `testing` package provides only `Error`, `Fatal`, `Log`, `Skip`. No `assert.Equal`, `assert.NotNil`. This is intentional:

- **Assertion libraries hide control flow** — `assert` might call `t.Fatal` or `t.Error` under the hood, but the signature suggests it returns normally. This confuses static analysis.
- **Explicit error handling** aligns with Go's philosophy elsewhere (`if err != nil`). `if got != want { t.Errorf(...) }` is explicit, not magical.
- **Less API surface** means fewer debates over `assert` vs `require` vs `assertThat`.

That said, community packages like `testify/assert` exist for those who prefer them. The standard library stays minimal.

### Test Files Live Alongside Code

Placing `_test.go` files in the same package (not a separate `tests/` directory) enables:
- Testing unexported functions (since `_test.go` is part of the package during compilation)
- Avoiding import cycles
- Immediate proximity between implementation and tests

This mirrors Go's "no separate header" design — everything you need is in one place.

### Table-Driven Tests Are Encouraged, Not Enforced

The `testing` package doesn't provide a special table-test syntax. Instead, Go's `struct` literals and slices make table-driven tests syntactically light. The `t.Run` addition (Go 1.7) gave table-driven tests subtest identity without external libraries. This composition of language features (not framework magic) is quintessential Go.

---

## 4. Competing Approaches: The "Context"

| Feature | Go | Java (JUnit 5) | Python (pytest) | Rust |
|---------|-----|----------------|------------------|------|
| Test discovery | Naming convention (`TestXxx`) | Annotations (`@Test`) | Function prefix (`test_`) | Attribute (`#[test]`) |
| Assertions | `if` + `t.Error` | `Assertions.assertEquals()` | `assert` statement | `assert_eq!` macro |
| Parameterized tests | Table-driven + `t.Run` | `@ParameterizedTest` | `@pytest.mark.parametrize` | `test_case` crate |
| Fuzzing | Built-in (Go 1.18+) | JQF (external) | `hypothesis` or `atheris` | `libfuzzer-sys` |
| Benchmarking | Built-in (`testing.B`) | JMH (external) | `pytest-benchmark` | `benchmark` crate |
| Parallelism | `t.Parallel()` per test | `@Execution(CONCURRENT)` | `pytest -n auto` | `cargo test -- --test-threads` |
| Test lifecycle | `TestMain` + `t.Cleanup` | `@BeforeEach`/`@AfterEach` | `setup_function`/`teardown_function` | `setup`/`teardown` attributes |

### Java's JUnit Philosophy

JUnit prioritizes **expressiveness through annotations**. The framework handles lifecycle, parameter injection, and assertions via extension points. This is powerful but creates a steep learning curve (e.g., "when does `@BeforeAll` run relative to `@Nested` classes?"). Go's tests are just functions — no magic, no surprises.

### Python's pytest Philosophy

pytest leverages Python's dynamic nature to rewrite `assert` statements with rich diagnostics. This is elegant but relies on **import hooks** that can fail in subtle ways with complex codebases. Go's static approach (`if got != want`) is more verbose but always works predictably.

### Rust's Cargo Test

Rust shares Go's built-in test command, but with a crucial difference: Rust tests **run in parallel by default** with no opt-in (`t.Parallel()` in Go requires explicit opt-in). Go's decision to require `t.Parallel()` forces developers to think about test isolation and prevents accidental sharing of mutable globals.

**Key Trade-off:** Go sacrifices magical assertion output (pytest) and deep annotation systems (JUnit) for **predictability and compilation-time correctness**. There's no test runtime configuration that can break — it's just Go code.

---

## 5. Common Mistakes: The "Gotchas"

### 1. Testing Unexported Functions Incorrectly

**Problem:** You move a function from `package foo` to `package bar`, but tests in `foo_test.go` still call it directly.

**Solution:** Either keep tests in the same package (no `_test` suffix on package clause) or redesign to export if truly public. For white-box testing, use `package foo` (not `package foo_test`).

```go
// Correct for testing internals
package math  // NOT package math_test

func TestAddInternal(t *testing.T) { /* can call unexported addInternal */ }
```

### 2. Ignoring `-race` in CI

**Problem:** Tests pass locally but fail under concurrency. Without `-race`, data races go undetected.

**Solution:** Always run `go test -race ./...` in CI. The overhead (~2-10x execution time) is worth catching heisenbugs.

### 3. Misusing `t.Fatal` in Goroutines

**Problem:** Calling `t.Fatal` (which stops the test) from a different goroutine panics because `t` is not safe for concurrent `Fatal`.

```go
// BROKEN
go func() {
    if err != nil {
        t.Fatal(err) // panic: test executed from non-test goroutine
    }
}()
```

**Solution:** Use `t.Error` and a synchronization mechanism (e.g., `sync.WaitGroup`), then `t.Fatal` after joining.

### 4. Benchmark Loop Folding

**Problem:** The compiler optimizes away code that has no observable side effects.

```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(1, 2) // may be optimized if Add is inlined and result unused
    }
}
```

**Solution:** Assign result to a global variable `var sink int` and set `sink = Add(1,2)`.

### 5. Not Using `t.Helper()`

**Problem:** Error messages point to your helper function line, not the test case line.

```go
func assertEqual(t *testing.T, got, want int) {
    if got != want {
        t.Errorf("got %d, want %d", got, want) // line 3
    }
}

func TestAdd(t *testing.T) {
    assertEqual(t, Add(2,3), 5) // failure reports line 3, not line 7
}
```

**Solution:** `t.Helper()` rewrites stack traces to show caller.

```go
func assertEqual(t *testing.T, got, want int) {
    t.Helper()
    if got != want {
        t.Errorf("got %d, want %d", got, want)
    }
}
```

### 6. Mutable Test Data Shared Across Subtests

**Problem:** Subtests run in parallel and mutate shared slice/map.

```go
func TestParse(t *testing.T) {
    sharedData := []string{"a", "b"} // shared across all t.Run closures
    
    for _, tc := range cases {
        t.Run(tc.name, func(t *testing.T) {
            t.Parallel()
            sharedData[0] = "mutated" // RACE
        })
    }
}
```

**Solution:** Copy or re-initialize per subtest, or avoid `t.Parallel()`.

---

## 6. Performance Considerations: The "Cost"

### Benchmark Overhead

The `testing.B` framework adds overhead of ~10-50ns per iteration for timing and loop control. For sub-nanosecond operations (e.g., integer addition), you must increase `b.N` artificially or use `b.RunParallel()`:

```go
func BenchmarkAdd(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            Add(1, 2)
        }
    })
}
```

### Compiler Optimizations and `-gcflags`

Benchmarks can mislead if compiler optimizations eliminate your code. Disable inlining and optimizations for microbenchmarks:

```bash
go test -bench=. -gcflags='-l -N'  # disable inlining and optimizations
```

However, production code will have optimizations enabled, so benchmark both ways.

### Test Execution Time vs. Compilation Time

For large packages, `go test` recompiles the entire package and its dependencies. This is faster than interpreted test runners (no runtime discovery overhead) but slower than incremental test runners like `pytest-watch`. Mitigate by:
- Structuring packages with limited dependencies
- Using build tags (`//go:build integration`) to separate fast unit tests from slow integration tests
- Running `go test -short` to skip expensive tests during development

### Memory Allocation in Tests

Tests can generate significant heap pressure, especially table-driven tests with large struct slices. Use `testing.AllocsPerRun` in benchmarks:

```go
func BenchmarkAllocs(b *testing.B) {
    reports := testing.AllocsPerRun(100, func() {
        var s strings.Builder
        s.WriteString("hello")
    })
    b.Logf("allocs per run: %f", reports)
}
```

### Fuzzing Performance

Fuzzing explores input space using coverage guidance. The fuzzing engine maintains a corpus and mutates inputs to maximize coverage. Expect:
- High CPU usage (100% of one core)
- Long running times (hours for deep code paths)
- Memory growth from corpus retention (use `-fuzzminimizetime`)

For CI, run fuzzing as a separate nightly job, not on every commit.

---

## 7. Best Practices: The "Idiomatic Way"

### 1. Table-Driven Tests with Subtests Always

Even for a single case, structure as a table with `t.Run`. This makes adding cases trivial and provides clear failure identifiers.

### 2. Use `t.Cleanup` Instead of Defer in TestMain

`t.Cleanup` attaches cleanup to the test's lifecycle, not the function's. It runs even if the test calls `t.Fatal`, and it runs after all subtests complete.

```go
func TestDB(t *testing.T) {
    db := connectDB()
    t.Cleanup(func() { db.Close() }) // guaranteed to run
    // test logic
}
```

### 3. Separate Fast from Slow Tests with Build Tags

```go
//go:build integration

package db_test

func TestRealDatabase(t *testing.T) {
    // requires running PostgreSQL
}
```

Run with: `go test -tags=integration ./...`

### 4. Test Your Test Helpers

Helper functions that assert conditions should themselves be tested for correctness. Use `t.Helper()` and consider testing edge cases in a separate test.

### 5. Use `testing/fstest` for File System Mocking

The `testing/fstest` package provides `MapFS` and `TestFS` to validate file system implementations against the `io/fs` contract.

```go
func TestMyFS(t *testing.T) {
    fsys := myfs.New()
    if err := fstest.TestFS(fsys, "file.txt", "sub/dir"); err != nil {
        t.Fatal(err)
    }
}
```

### 6. Benchmark Both Happy Path and Worst Case

Include benchmarks for:
- Small inputs (e.g., 10-element slice)
- Large inputs (10,000-element slice)
- Edge cases (empty, nil, single element)

Use `b.ResetTimer()` before the loop to exclude setup cost.

### 7. Test Unexported Functions via `export_test.go`

A Go idiom for white-box testing without exposing internals publicly:

```go
// math/export_test.go
package math

var SumInternal = sumInternal // expose unexported function for testing
```

Then in `math_test.go` you can call `SumInternal()` as if exported.

---

## 8. Examples

### Example 1: Concurrent Cache with Race Testing

```go
// cache.go
package cache

import "sync"

type Cache struct {
    mu    sync.RWMutex
    items map[string]string
}

func NewCache() *Cache {
    return &Cache{items: make(map[string]string)}
}

func (c *Cache) Set(key, value string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = value
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    val, ok := c.items[key]
    return val, ok
}
```

```go
// cache_test.go
package cache

import (
    "sync"
    "testing"
)

func TestCacheConcurrency(t *testing.T) {
    c := NewCache()
    var wg sync.WaitGroup
    const goroutines = 100

    for i := 0; i < goroutines; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            key := "key"
            c.Set(key, "value")
            if val, ok := c.Get(key); !ok || val != "value" {
                t.Errorf("goroutine %d: got %q, ok=%v", id, val, ok)
            }
        }(i)
    }
    wg.Wait()
}
```

Run with `go test -race` to verify no data races.

### Example 2: HTTP Client with Test Server

```go
func TestHTTPClient(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte(`{"status":"ok"}`))
    }))
    t.Cleanup(server.Close)

    client := &http.Client{Timeout: 5 * time.Second}
    resp, err := client.Get(server.URL + "/health")
    if err != nil {
        t.Fatal(err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        t.Errorf("expected 200, got %d", resp.StatusCode)
    }
}
```

### Example 3: Benchmark for Map Lookup Strategies

```go
func BenchmarkMapStringKey(b *testing.B) {
    m := make(map[string]int)
    for i := 0; i < 1000; i++ {
        m[string(rune(i))] = i
    }

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _ = m["key500"]
    }
}

func BenchmarkMapIntKey(b *testing.B) {
    m := make(map[int]int)
    for i := 0; i < 1000; i++ {
        m[i] = i
    }

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _ = m[500]
    }
}
```

### Example 4: Fuzzing a JSON Parser

```go
func FuzzJSONUnmarshal(f *testing.F) {
    f.Add(`{"name":"test"}`)
    f.Add(`[]`)

    f.Fuzz(func(t *testing.T, data []byte) {
        var v any
        if err := json.Unmarshal(data, &v); err != nil {
            t.Skip() // invalid JSON is not a bug
        }
        // Property: marshaling should produce equivalent structure
        roundTripped, err := json.Marshal(v)
        if err != nil {
            t.Fatalf("marshal failed after unmarshal: %v", err)
        }
        var v2 any
        if err := json.Unmarshal(roundTripped, &v2); err != nil {
            t.Fatalf("second unmarshal failed: %v", err)
        }
        // Compare v and v2 (use reflect.DeepEqual)
        if !reflect.DeepEqual(v, v2) {
            t.Errorf("round-trip mismatch: got %#v, want %#v", v2, v)
        }
    })
}
```

---

## 9. Summary & Exercises

### Summary

- **`go test`** is built-in, discovers `*_test.go` files, and runs `TestXxx` functions with no external frameworks.
- **Table-driven tests** with `t.Run` are the idiomatic pattern for multiple cases and parallel execution.
- **Fuzzing** (Go 1.18+) automatically explores edge cases and generates minimized crash inputs.
- **Benchmarks** use `testing.B` with adaptive `b.N` scaling; guard against compiler optimizations with result sinks.
- **Common pitfalls** include mutable shared data, forgetting `t.Helper()`, and ignoring the race detector.
- **Philosophy:** Built-in testing reduces friction, explicit assertions avoid magic, and composition over framework inheritance.

### Exercises

#### Exercise 1: Build a Thread-Safe Cache with Concurrency Tests

Implement a `SafeCache` that supports `Get`, `Set`, `Delete`, and `Len`. Then write:
- A table-driven test for basic operations
- A parallel test that runs 1000 concurrent goroutines performing mixed operations, verifying final state consistency
- A benchmark comparing your implementation with `sync.Map`

**Constraints:** Use `go test -race` to verify correctness. Your tests should complete within 2 seconds.

#### Exercise 2: Fuzz a Custom String Parser

Write a function `ParseKeyValue(s string) (key, value string, err error)` that splits on the first `=` character, trims spaces, and rejects empty keys or values. Then:
- Create a fuzz test that generates arbitrary strings
- Verify the property: parsing a valid string `"k=v"` and then reconstructing `k + "=" + v` returns the original
- Run the fuzzer for 10 seconds and report any crashes

**Challenge:** Modify the fuzzer to seed from a file containing common malformed inputs (e.g., `"a=b=c"`, `" = "`, `"a="`).

#### Exercise 3: Profile a Slow Test Suite

Given a package with intentionally slow tests (simulate by calling `time.Sleep(5*time.Millisecond)` in each test), use `go test -benchmem -cpuprofile=cpu.out` and `go tool pprof` to identify the bottleneck. Then:
- Refactor the tests to use `t.Parallel()` where safe
- Use `testing/quick` or `fstest` to replace I/O mocks
- Measure the speedup factor

**Deliverable:** A short analysis (3-5 sentences) explaining which optimization gave the largest improvement and why.
