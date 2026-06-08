## Appendix B: Go Doc Deep Dive

Documentation in Go is not an afterthought bolted on by external tools; it is a first‑class citizen woven into the language specification, the standard library, and the development workflow. The `go doc` command, together with the conventions for writing doc comments, forms a documentation system that prizes simplicity, readability, and zero‑cost maintenance. This appendix shows how to wield that system effectively, from day‑to‑day usage to the design philosophy that makes it tick.

---

### 1. Basic Usage

The `go doc` command prints documentation for packages, types, functions, and methods. Its power lies in the fact that it works on any Go source code—standard library, third‑party modules, or your own project—without the need to generate, build, or render anything separately.

**Viewing package documentation**

```bash
# Documentation for the current package
go doc

# A specific package in the standard library
go doc strings

# A package in the current module (qualified by import path)
go doc mymodule/internal/auth

# The package declaration and exported API of a local directory
go doc .
```

**Drilling down to a symbol**

```bash
# A function
go doc strings.Contains

# A type and all its methods
go doc net/http.Handler

# A method (note the dot after the receiver type)
go doc net/http.Handler.ServeHTTP

# An exported field of a struct
go doc time.Time.Month
```

**Flags that shape the output**

- `-all` : show all documentation, including unexported symbols and methods, and full package text.
- `-src` : display the source code of the symbol instead of just the doc comment.
- `-short` : print only the one‑line summary for each symbol.
- `-C dir` : change to `dir` before running (useful in scripts or from outside the module).

```bash
go doc -all strings.Builder       # unexported fields and methods too
go doc -src sync.Mutex            # see the implementation inline
go doc -short net/http            # scan the API quickly
```

`go doc` also understands type parameters. For a generic function `func Max[T cmp.Ordered](a, b T) T` you can write:

```bash
go doc cmp.Ordered                # the constraint
go doc mypkg.Max                  # the generic function
```

The output includes the full signature with type parameters when they are present.

---

### 2. Under the Hood

`go doc` is essentially a thin wrapper around two standard library packages: `go/doc` and `go/ast`. When you run the command, the tool:

1. Resolves the package path (using the module cache or the local file system).
2. Parses the Go source files with `go/parser`, producing an abstract syntax tree.
3. Runs `go/doc.New` to create a `*doc.Package` value. This step filters out unexported names (unless `-all` is given), collects the doc comments associated with each exported identifier, and resolves the relationships among types, methods, and embedded fields.
4. Formats the resulting documentation as plain text and writes it to stdout.

The critical insight is that **doc comments are structured by their position in the source**, not by special tags or markup. The `go/doc` package uses a simple heuristic: the comment group immediately preceding a declaration belongs to that declaration. There is no separate “doc generation” phase—the very act of compiling the code is enough to extract the documentation.

**How `go doc` handles inline examples**

When you write an `Example` function in a `_test.go` file, it is discovered by `go/doc` and associated with the package, type, or function it illustrates. `go doc` then prints the example’s body as pre‑formatted text. The `go test` tool also executes those examples and verifies their `// Output:` comments, seamlessly merging documentation with testing.

**The caching layer**

Because `go doc` uses the same module‑aware machinery as `go build`, it benefits from the build cache. Once a package has been downloaded and parsed, subsequent `go doc` invocations for the same version are nearly instantaneous.

---

### 3. Why This Design?

The Go team deliberately rejected the kind of documentation system that requires a separate processing step, dedicated markup language, or external toolchain. The reasoning rests on three pillars.

**1. Simplicity and zero ceremony**

In many ecosystems, documenting a function feels like writing a small program: you must learn Javadoc tags (`@param`, `@return`), Python’s reStructuredText directives, or Rust’s `///` attributes and markdown. Go’s message is, “You already know how to write comments; just put them in the right place.” There is no special syntax to memorise. This lowers the barrier to writing documentation, which in turn increases the probability that it actually gets written.

**2. Source code is the single source of truth**

Because documentation lives inside `.go` files, it cannot rot independently of the code. When a function signature changes, the comment is right there, inviting the author to update it. Separate documentation files (e.g., XML, Markdown, or Sphinx pages) tend to drift because they are disconnected from the code they describe.

**3. Uniformity across the ecosystem**

Every Go package, from the smallest library to the largest monorepo, is documented with the same tool and the same conventions. `go doc` works on any importable package. This uniformity fosters a culture where engineers expect to be able to `go doc` anything they `go get`, and where contributions that lack proper doc comments are considered incomplete.

These principles are a natural extension of Go’s overall philosophy: **simplicity over complexity, and tools over frameworks.**

---

### 4. Competing Approaches

Comparing Go’s documentation model with those of other languages highlights the trade‑offs involved.

- **Java / Javadoc**: Javadoc uses block comments (`/** ... */`) with a rich set of `@` tags. It generates HTML and can include cross‑references. This is powerful but heavy: the developer must learn the tag vocabulary, and the toolchain is separate from the compiler. Go’s approach achieves a similar level of structured output (via `go doc` and `pkg.go.dev`) with far less syntactic noise.

- **Python (docstrings)**: Python attaches string literals to modules, classes, and functions. Tools like Sphinx can extract them, but the reStructuredText or Markdown formatting is optional and inconsistent across projects. Go’s doc comments are simpler and uniformly available through `go doc`, whereas Python’s `help()` is often less accessible unless the developer has explicitly written docstrings.

- **Rust**: Rust’s `///` and `//!` doc comments support full CommonMark Markdown, with the compiler even running code snippets as tests. This is a more feature‑rich system, but it forces developers to learn Markdown semantics and often leads to doc comments that are less readable in plain text. Go’s philosophy favours comments that read just as well in a terminal as on a web page.

- **JavaScript / JSDoc**: JSDoc is entirely optional and relies on special `/** */` annotations. The tooling is external, and many projects simply don’t use it. Go makes documentation a natural, integrated part of the development experience, not an opt‑in decoration.

- **C# / XML documentation**: Similar to Javadoc, C# uses XML tags inside `///` comments. This approach is tightly coupled with Visual Studio’s IntelliSense, but the XML is verbose and awkward to read in source form. Go’s plain‑text comments are far less intrusive.

The common thread is that Go **sacrifices feature depth for universal adoption and readability of the source**. In exchange, the entire ecosystem benefits from a single, zero‑effort documentation view.

---

### 5. Common Mistakes

Even experienced engineers make subtle errors when writing Go doc comments. The most frequent ones are:

- **Misplacing the package comment**: The package comment must be placed immediately before the `package` declaration with no blank line between them. If a blank line separates them, the comment is treated as the documentation for the next declaration (usually the first import or function), and the package appears undocumented.

```go
// WRONG: blank line kills the package doc

// Package mypkg does wonderful things.
package mypkg
```

Correct version:

```go
// Package mypkg does wonderful things.
package mypkg
```

- **Using block comments for package documentation**: While legal, `/* */` comments are error‑prone because subsequent blank lines can sever the association. The `go/doc` package prefers `//` line comments. The standard practice is to use `//` for package doc, and reserve `/* */` for longer internal notes that are not documentation.

- **Redundant doc comments**: Comments like `// Foo returns foo.` add zero information. The convention is to start with the name of the symbol and then describe *what it does*, *why*, and *how*. A good doc comment answers questions that are not obvious from the signature alone.

- **Forgetting to document exported identifiers**: The Go compiler does not warn about undocumented exports, but `go vet` can be configured to do so, and many CI pipelines enforce it. A package with half‑documented API is frustrating to use.

- **Documenting the implementation instead of the contract**: Doc comments should be written for the consumer. Details about internal algorithms, locking order, or memory layout belong in ordinary `//` comments inside the function body, not in the exported doc.

- **Incorrect use of indentation for code examples**: Code blocks inside a doc comment must be indented by one additional tab (or space) relative to the surrounding text. Failing to indent correctly will render the code as plain text, destroying readability.

```go
// Split slices s into all substrings separated by sep and returns a slice of
// the substrings between those separators.
//
//   s := "a,b,c"
//   parts := strings.Split(s, ",")
//   fmt.Println(parts) // [a b c]
```

- **Neglecting examples in `_test.go` files**: Many developers write excellent doc comments but omit runnable examples. `Example` functions serve double duty: they are shown by `go doc` and are executed as tests. Their absence is a missed opportunity to demonstrate intended usage and safeguard against regression.

---

### 6. Performance Considerations

Documentation itself has no runtime cost, but there are compile‑time and tool‑performance aspects worth noting.

- **Compilation overhead**: Doc comments are part of the source code and are parsed by the compiler, but they do not affect generated code size or binary performance. Extremely large doc comments (e.g., a 10 KB package overview) can slow down parsing by a negligible amount. In practice, this is not a concern.

- **`go doc` latency**: `go doc` benefits from the module cache. The first invocation for a remote package downloads the module; subsequent calls are local and fast. For local packages, parsing is very quick, but on a monorepo with thousands of files, you may notice sub‑second delays. Using `-C` to pin to a specific directory avoids walking the entire module graph.

- **`pkg.go.dev` rendering**: The documentation site for the Go ecosystem runs `go/doc` on every package version and caches the result. Heavy use of code blocks and large examples has no effect on your users’ experience. However, excessive use of the new doc comment links (e.g., a thousand `[SomePkg]` references) can slow rendering marginally; keep it pragmatic.

- **Memory impact**: The `go/doc` package builds in‑memory data structures for all documented symbols. For packages with tens of thousands of exported identifiers (rare), this may cause a memory spike in `go doc` or in IDE tools that index doc comments. Again, this is almost never a real‑world problem.

In summary, the performance profile of Go’s documentation system is a non‑issue for the overwhelming majority of projects.

---

### 7. Best Practices

Idiomatic Go documentation follows a set of conventions that have evolved with the language. The definitive guide is the official [Go Doc Comments](https://tip.golang.org/doc/comment) page (updated for Go 1.19+), which codified several enhancements while preserving backward compatibility.

**Package comment**

- Place it in a file named `doc.go` (convention, not requirement). That file may contain nothing but the package comment and the `package` statement.
- Begin with `// Package <name> ...` and provide a one‑paragraph summary.
- If more detail is needed, add a blank line and continue with additional paragraphs.
- Use indentation for pre‑formatted code snippets.
- End with a period; use complete sentences.

**Exported symbols**

- Start each doc comment with the name of the symbol, followed by a sentence fragment or full sentence that describes its purpose: `// TrimSpace removes leading and trailing whitespace from s.`
- For functions with multiple return values, describe what each value means.
- Document any concurrency guarantees (`// It is safe for concurrent use.` or `// The caller must hold the mutex.`).
- Do not list the parameter names unless the meaning is non‑obvious. The signature already displays them.

**Links, lists, and headings (Go 1.19+)**

The `go/doc` package now recognises a limited markup syntax that renders beautifully on `pkg.go.dev` while remaining legible as plain text in the terminal.

- **Links**: `[pkg.Name]` links to the named symbol within the current module. `[text]: https://example.com` creates an external reference.
- **Headings**: A line starting with `#` followed by a space is treated as a heading. However, `go doc` in the terminal simply displays it as‑is, so use headings sparingly.
- **Lists**: Lines that begin with a bullet character (`-`, `*`, `+`) or a number followed by a period are recognised as list items.

For maximum compatibility with both `go doc` and `pkg.go.dev`, prefer the plain‑text style and use the new syntax only when it adds clarity.

**Runnable examples**

- Name an example function `Example`, `Example_type`, `Example_type_method`, or `Example_suffix`.
- The function should print to stdout. Add an `// Output:` comment to specify the expected output; this turns the example into a test.
- Examples are automatically included in `go doc` output, making them the most effective form of documentation.

```go
func ExampleSplit() {
    s := "a,b,c"
    parts := strings.Split(s, ",")
    fmt.Println(parts)
    // Output: [a b c]
}
```

**Organising documentation for large packages**

- Use `doc.go` for the package overview.
- For complex types, consider writing the main type comment directly above the type declaration and placing longer usage narratives in a separate `_test.go` file as example functions.
- Avoid burying crucial information in unexported helper comments that users will never see.

---

### 8. Examples

Below is a complete mini‑package that demonstrates idiomatic documentation. All doc comments follow the guidelines.

```go
// Package bank provides a thread‑safe representation of a simple bank account.
//
// It is designed as a teaching tool for concurrency, not for production use.
// The exported API is intentionally minimal.
package bank

import "sync"

// Account represents a bank account with a balance.
// The zero value is an empty account with a zero balance.
// It is safe for concurrent use.
type Account struct {
    mu      sync.RWMutex
    balance int64 // balance in the smallest currency unit (e.g., cents)
}

// Deposit adds amount to the account.
// Amount may be negative, in which case the balance decreases.
func (a *Account) Deposit(amount int64) {
    a.mu.Lock()
    defer a.mu.Unlock()
    a.balance += amount
}

// Balance returns the current balance.
func (a *Account) Balance() int64 {
    a.mu.RLock()
    defer a.mu.RUnlock()
    return a.balance
}
```

Place an example in a `_test.go` file:

```go
func ExampleAccount() {
    acc := bank.Account{}
    acc.Deposit(100)
    acc.Deposit(-30)
    fmt.Println(acc.Balance())
    // Output: 70
}
```

Now run:

```bash
go doc
# Output:
# Package bank provides a thread‑safe representation of a simple bank account.
# ...

go doc bank.Account
# Output:
# type Account struct { ... }
#     Account represents a bank account with a balance.
#     ...

go doc bank.Account.Deposit
# Output:
# func (a *Account) Deposit(amount int64)
#     Deposit adds amount to the account.
```

The output is clean, informative, and immediately useful—no separate command required.

---

### 9. Summary & Exercises

**Recap**

- Go’s documentation system revolves around `go doc` and plain‑text comments placed directly before declarations.
- Package documentation lives in a `doc.go` file; every exported identifier should have a concise, consumer‑focused comment.
- Runnable examples in `_test.go` files are the gold standard for demonstrating API usage.
- The system embodies Go’s philosophy of simplicity, uniformity, and source‑code‑as‑truth.

**Exercises**

1. **Document an existing package**
   Take a small open‑source Go library that lacks comprehensive documentation. Write a `doc.go` file with a package overview, add doc comments for every exported type and function, and provide at least two `Example` functions. Run `go doc` on the package and verify the output. Run `go test` to ensure the examples are correct.

2. **Design a documentation‑driven API**
   Before implementing anything, write the package comment and the doc comments for the public API of a key‑value cache with TTL expiration. Include example functions that show how to set, get, and delete entries, and how the cache handles expiry. Share the documentation with a colleague and ask them to implement the package solely from the doc. Reflect on what was missing or ambiguous.

3. **Audit a large codebase**
   In a production Go service, run `go doc -all` on the project’s root package and identify at least five exported symbols that lack documentation or whose documentation is misleading. Create a plan to fill the gaps, then implement it. Measure whether the team’s onboarding time improves after the documentation is in place.

When you embrace `go doc` as a first‑class tool, you stop seeing documentation as a chore and start seeing it as a conversation with the engineers who will read your code—tomorrow, next month, or in five years.
