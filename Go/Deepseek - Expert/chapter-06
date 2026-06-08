## Chapter 6: Control Flow

Go’s control flow constructs are deliberately minimal. There is only one loop keyword, `switch` is less error-prone than in C-derived languages, and `if` can accept a short statement that scopes a variable to the block. This chapter dissects the syntax, the runtime mechanics, and the design rationale behind each piece.

---

### 1. Basic Usage

#### `if` with a Short Statement

The `if` statement can execute a simple statement before the condition. This statement’s scope is the `if` block (including any `else if` and `else` blocks). It is the canonical way to handle errors and keep the happy path unindented.

```go
func fetchUser(id string) (User, error) {
    if id == "" {
        return User{}, errors.New("id must not be empty")
    }
    // ...
}

func process(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return fmt.Errorf("opening %s: %w", path, err)
    }
    defer f.Close()
    // use f
    return nil
}
```

You can also use a short statement to limit the lifetime of a temporary variable:

```go
if v, ok := cache.Load(key); ok {
    // v and ok are only visible here
    return v.(Data), nil
}
```

#### `switch` — Expression and Type Switches

Go’s `switch` does **not** fall through by default. Cases break implicitly. To fall through, you must explicitly write `fallthrough`, which is rarely needed. The evaluated expression can be any comparable type, and a switch with no expression is equivalent to `switch true` — a clean way to replace long `if‑else if` chains.

```go
switch statusCode {
case 200, 201, 204:
    handleSuccess()
case 301, 302:
    handleRedirect()
case 404:
    handleNotFound()
default:
    handleError()
}
```

A tagless switch:

```go
switch {
case x > 0:
    sign = 1
case x < 0:
    sign = -1
default:
    sign = 0
}
```

A type switch asserts the dynamic type of an interface value:

```go
func describe(v any) string {
    switch v := v.(type) {
    case int:
        return fmt.Sprintf("int %d", v)
    case string:
        return fmt.Sprintf("string %q", v)
    default:
        return fmt.Sprintf("%T", v)
    }
}
```

#### `for` — The Only Loop

`for` subsumes C’s `while` and `do‑while`. There are three forms:

1. **Classic C-style:** `for init; condition; post { }`
2. **Condition-only (while):** `for condition { }`
3. **Infinite:** `for { }` — break out with `break` or `return`.

```go
for i := 0; i < 10; i++ {
    // ...
}

for scanner.Scan() {
    // process line
}

for {
    // wait for a message
    select {
    case msg := <-ch:
        // handle
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

#### `range` — Iterating Collections

`range` returns different values depending on the data type:

- **Array/slice:** `(index int, value T)` – `value` is a **copy** of the element.
- **String:** `(index int, runeValue rune)` – index is the byte offset, value is the decoded `rune`.
- **Map:** `(key K, value V)` – iteration order is non-deterministic.
- **Channel:** `(value T)` – reads until the channel is closed.

```go
// slice
for i, v := range items {
    fmt.Println(i, v)
}

// string: careful – see “Common Mistakes”
for pos, r := range "Hello, 世界" {
    fmt.Printf("byte %d: %c\n", pos, r)
}

// map
for k, v := range scores {
    fmt.Println(k, v)
}

// channel
for msg := range messages {
    process(msg)
}
```

#### Labels and `goto`

Labels mark a statement and can be targeted by `break`, `continue`, or `goto`. `break label` and `continue label` are used to control outer loops. `goto` can jump anywhere within the same function, but it must not skip variable declarations.

```go
outer:
for i := 0; i < 10; i++ {
    for j := 0; j < 10; j++ {
        if i*j > 50 {
            break outer   // exits both loops
        }
    }
}
```

`goto` is typically reserved for generated code, state machines, or cleaning up in error paths where multiple resources need releasing.

---

### 2. Under the Hood

Go’s compiler (`gc`) transforms control flow into an intermediate representation (SSA) and then to machine code. Here’s what happens behind the scenes.

#### `if` Short Statement

The short statement is simply a node in the AST whose execution precedes the condition evaluation. The compiler allocates the variable on the stack (or heap if it escapes) and manages its lifetime to end at the block’s closing brace. No hidden overhead exists; it is equivalent to declaring the variable immediately before the `if` but with tighter scoping rules.

#### `for` Loop Implementation

All `for` variants are lowered to a single loop construct with a conditional branch. The compiler performs **bounds check elimination** on indexed access within loops when it can prove the index stays within the slice/array bounds. For `range` over slices and arrays, the compiler generates efficient pointer arithmetic. For `range` over a string, each iteration decodes a UTF‑8 sequence, advancing a byte pointer. The `range` over a map calls the runtime hash table iterator, which picks a random bucket to start from — guaranteeing the non-deterministic iteration order.

`for range` over a slice assigns a **copy** of each element to the loop variable. This means that the address of the loop variable is reused across iterations. If you take its address, you will get the same pointer each time, which trips up many newcomers (see Common Mistakes).

#### `switch` Implementation

The compiler classifies a `switch` statement into one of several forms:

- **Expression switch with integer cases in a dense range** → compiled to a jump table (O(1) dispatch), similar to C’s `switch`.
- **Sparse cases or non‑integer** → a binary search tree of comparisons, or a chain of if‑else, depending on number of cases.
- **Tagless switch** → exactly a chain of `if‑else if`; no magic.
- **Type switch** → the runtime inspects the interface’s dynamic type (`iface.typ`), then uses a hash table of type descriptors to branch.

Because there is no implicit fallthrough, the compiler can freely reorder branches that have no side effects, though in practice it follows source order.

#### Labels and `goto`

Labels are resolved at compile time. `goto` is a direct jump within a function’s control flow graph. The runtime does not get involved; the jump simply alters the instruction pointer. The compiler enforces that `goto` does not skip the initialisation of any variable declared in the path. If you need to jump over a declaration, you must enclose it in a block — this restriction prevents reads of uninitialised memory.

---

### 3. Why This Design?

The control flow design stems from Go’s north star: **simplicity and clarity**.

#### Only One Loop Construct

Languages like C, C++, Java, and Python provide multiple looping keywords (`while`, `do‑while`, `for`, `foreach`). Go’s designers observed that `for` can express all of them with less surface area. A single keyword reduces the cognitive load of reading unfamiliar code and makes tooling simpler. The argument “why learn two ways to write a loop?” resonates with Go’s “less to learn” philosophy.

#### `switch` Without Implicit Fallthrough

C’s fallthrough-by-default caused countless bugs and led to coding standards demanding a `break` comment when fallthrough was intentional. Go reverses the default: you opt into fallthrough with an explicit keyword. This eliminates an entire class of errors. Additionally, a `switch` with no expression provides a cleaner alternative to deep `if‑else if` chains; it aligns the conditions vertically and makes them scannable.

#### `if` With Short Statement

This pattern elegantly ties a setup call (like a map lookup or error‑returning function) to the conditional. It avoids leaking temporary variables into the outer scope and encourages the “happy path” to remain at the outermost indentation level. This is a deliberate nudge toward error handling that reads like a linear story.

#### `range` Returns Copies

Returning a copy of each element from a slice/array prevents accidental sharing of loop variable addresses, which was a notorious bug in languages that reuse a reference. Go’s approach forces the developer to be explicit if they want a pointer: they must index the original slice (`for i := range items { p := &items[i] }`). This trade‑off slightly penalises large struct copies, but Go expects you to use pointers in that case.

---

### 4. Competing Approaches

#### C / C++ / Java

All three languages have multiple loop constructs and C‑style `switch` with fallthrough. Go’s `switch` reduces errors. C++ and Java have `for`‑each loops that implicitly reference or copy; Java’s enhanced `for` always gives a reference for objects, which can lead to unexpected mutation of the original collection if not careful. Go’s copy semantics are safer by default.

C’s `goto` is unrestricted and often used for cleanup; Go’s `defer` largely removes the need for error‑path `goto`. However, Go retains `goto` for those rare cases where jumping is the clearest solution, e.g., generated state machines.

#### Python

Python uses `for … in` over iterables, `while`, and has no `switch` until Python 3.10 (`match`). Go’s `range` is conceptually similar but statically typed and compiles to tight loops. Python’s iteration model is based on an iterator protocol, which can be slower and heap‑allocates iterator objects. Go’s `range` over slices avoids any heap allocation.

#### Rust

Rust has `loop`, `while`, `for`, and `match`. `match` is exhaustive by default, unlike Go’s `switch`, which requires a `default` case to be exhaustive. Rust’s `for` uses iterators, enabling zero‑cost abstractions with adaptors (map, filter). Go’s `for`/`range` is deliberately simple; complex transformations are written as explicit loops, which can be more verbose but are arguably clearer. Rust’s ownership system eliminates the loop variable address reuse problem that Go handles via copy semantics.

#### JavaScript

JavaScript’s `for` variants (`for`, `for…in`, `for…of`, `while`, `do…while`) are numerous. `for…in` iterates over enumerable properties (and includes prototypes), while `for…of` works with iterables. Go’s single `range` is unambiguous: on arrays/slices you get index and value; on maps you get key and value; on strings you get byte offset and rune. No surprises from prototype pollution.

---

### 5. Common Mistakes

#### Taking the Address of the Range Variable

The loop variable in `for _, v := range slice` is reused each iteration. Its address remains constant. Taking `&v` yields the same pointer pointing to successive copies.

```go
var out []*int
for _, v := range []int{1, 2, 3} {
    out = append(out, &v) // WRONG: all elements point to the same address, last value
}
```

**Fix:** Use the slice index.

```go
for i := range data {
    out = append(out, &data[i])
}
```

#### Modifying the Collection While Iterating

Adding or deleting elements from a map during iteration is allowed but the iteration behavior is undefined; the runtime may skip new entries or iterate them. For slices, the length used by `range` is evaluated once at the start, so appending to the slice will not be reflected in the iteration.

#### `range` Over a String: Byte vs. Rune

`for i, v := range s` iterates over Unicode code points (`rune`), with `i` being the byte offset. A simple `for i := 0; i < len(s); i++` accesses individual bytes, which may split a multi‑byte character. Mistaking the two yields garbled text.

#### Shadowing Variables in `if` Short Statement

If you reuse a variable name in the short statement, the new variable shadows the outer one inside the block, but not in the `else`. This can lead to subtle bugs.

```go
var err error
if x, err := fetch(); err != nil {
    // err is the new variable
}
// err here is the outer one, still nil
```

Use distinct names or avoid shadowing when the outer variable must be updated.

#### Using `break` Without a Label in Nested Loops

A plain `break` terminates only the innermost `for`/`switch`/`select`. To break an outer loop, you need a label. A common mistake is expecting `break` inside a `switch` inside a `for` to exit the loop — it only exits the `switch`. Use `break loopLabel`.

#### `goto` Over Variable Declarations

The compiler forbids jumping over a variable declaration. For example:

```go
goto skip
v := 42 // compile error: goto skip jumps over declaration of v
skip:
```

You must enclose the declaration in its own block if you need to jump past it.

---

### 6. Performance Considerations

#### Loop Overhead

Go’s `for` loops compile to tight machine code. The `range` form over a slice can be as fast as indexed access, provided the loop body does not force bounds checks. The compiler performs bounds check elimination: if the loop is over the entire slice and uses the index only for accessing the slice, the checks are removed. A `range` over a string does UTF‑8 decoding per iteration; if you only need bytes, use a byte slice conversion `[]byte(s)` or a standard `for i` loop.

#### `range` on Maps

Map iteration is O(n) and involves random access to hash table buckets. It is slower per element than slice iteration. There is no way to control iteration order, and each iteration goes through the full bucket sequence. Avoid tight map iteration in performance‑critical code; consider an ordered slice of keys if order matters.

#### `switch` Dispatch

For a dense integer `switch`, the compiler will use a jump table, resulting in O(1) dispatch with a few cycles overhead. Sparse or non‑integer cases degenerate to a series of comparisons. If you have more than ~5 cases, `switch` is usually faster than an equivalent `if‑else if` chain because the compiler can optimise the branching. For string switches, the compiler builds a perfect hash table when possible, delivering constant‑time dispatch.

#### Escape Analysis in `range`

If the loop variable escapes (e.g., you store its address in a heap‑allocated slice), the copy of each element must be moved to the heap, increasing GC pressure. In such cases, prefer the index‑based loop and taking the address of the original slice element, which may avoid an extra allocation if the element is already on the heap.

#### `goto` and Code Generation

`goto` itself adds no runtime cost; it’s a simple jump. However, its misuse can prevent compiler optimisations like inlining or constraining the control flow graph, potentially harming performance indirectly. Stick to structured loops.

---

### 7. Best Practices

- **Use `if` with short statement for errors and map lookups.** It limits variable scope and reduces nesting.
- **Prefer `switch` over long `if‑else if` chains**, especially when evaluating the same variable. A tagless switch is idiomatic for multi‑condition branching.
- **Use `for range` for slice and map iteration** unless you need the index for something specific.
- **When you need a pointer to a slice element, use `for i := range s { p := &s[i] }`.** Never take the address of the loop value variable.
- **For strings, always use `for _, r := range s` to iterate runes.** If you need byte‑level access, convert to `[]byte` first to make the intent explicit.
- **Avoid `goto` in hand‑written code.** Legitimate uses are very rare: breaking out of deeply nested loops (use a label with `break` instead), or generated code such as parser automata. `defer` covers most cleanup paths.
- **Label only outer loops that need `break` or `continue` from inner blocks.** Don’t label everything; it clutters the code.
- **Prefer `for condition {}` over `for { if !condition { break } }`** for readability.
- **Use a `return` inside a `for` loop when you’ve found what you need.** Avoid state flags that get checked after the loop.

---

### 8. Examples

**Example 1: Finite State Machine with `switch` and `for`**

```go
type State int
const (
    StateStart State = iota
    StateWord
    StateEnd
)

func countWords(s string) int {
    state := StateStart
    count := 0
    for _, r := range s {
        switch state {
        case StateStart:
            if !unicode.IsSpace(r) {
                state = StateWord
                count++
            }
        case StateWord:
            if unicode.IsSpace(r) {
                state = StateStart
            }
        }
    }
    return count
}
```

**Example 2: Breaking out of a Nested Loop with a Label**

```go
func findPair(matrix [][]int, target int) (int, int, bool) {
    for i, row := range matrix {
        for j, val := range row {
            if val == target {
                return i, j, true
            }
        }
    }
    return 0, 0, false
}
```

While a `return` is cleaner here, a label is appropriate when you need to break from an inner loop to the outer without returning:

```go
outer:
for _, row := range matrix {
    for _, val := range row {
        if val == target {
            rowMatch = true
            break outer
        }
    }
}
```

**Example 3: Retry with Exponential Backoff Using `for` and `time.Sleep`**

```go
func retry(ctx context.Context, fn func() error) error {
    const maxRetries = 5
    backoff := 100 * time.Millisecond
    for i := 0; ; i++ {
        err := fn()
        if err == nil {
            return nil
        }
        if i >= maxRetries {
            return fmt.Errorf("max retries exceeded: %w", err)
        }
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(backoff):
            backoff *= 2
        }
    }
}
```

---

### 9. Summary & Exercises

**Summary**

Go’s control flow is intentionally sparse. `if` can bind variables tightly to the conditional block. `switch` is safe, expressive, and compile‑time optimised. `for` is the sole loop keyword, covering all common patterns without syntactic sugar. `range` iterates over slices, maps, strings, and channels with clear semantics, though its value semantics demand caution when taking addresses. Labels and `goto` exist for the rare cases where structured code becomes contorted, but they should not be your first reflex.

**Exercises**

1. **Implement a thread‑safe LRU cache iterator.**
   Given an LRU cache with a mutex, write a method `Range(f func(key, value any) bool)` that iterates over all entries while holding the lock. The function `f` should be able to stop iteration early by returning `false`. Use a `for` loop and a `break` to respect the early termination. Ensure the lock is released even if `f` panics (hint: `defer`).

2. **Build a lexer for a tiny configuration language.**
   The language consists of key‑value pairs separated by `=`, one per line, with optional whitespace. Keys are identifiers (`[a-zA-Z_][a-zA-Z0-9_]*`), values can be strings (double‑quoted, with backslash escapes) or integers. Use a `for` loop over runes and a `switch` on the current state to tokenize the input. Handle escape sequences correctly. Return an error on malformed input. This exercise forces you to combine `range` over a string, `switch` for state transitions, and `continue`/`break` for control.

3. **Write a function that flattens a nested integer slice of arbitrary depth using a label.**
   `flatten(nested [][]int) []int` is straightforward, but implement a general `flattenDeep(data []any) ([]int, error)` where `data` can contain `int` values or nested `[]any`. Use a depth‑first traversal with an explicit stack and a `for` loop. Use a label to break out of the loop when the result slice reaches a limit of, say, 10,000 elements (to prevent runaway memory). This demonstrates the limited but legitimate use of a label.
