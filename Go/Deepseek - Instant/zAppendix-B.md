## Appendix B: Go Doc Deep Dive

**Using `go doc` and Writing Documentation for `go doc`**

Documentation in Go is not an afterthought—it is a **first-class citizen** built directly into the toolchain. Unlike many ecosystems where docs live in separate wikis or comment generators, Go treats documentation as code. The `go doc` command transforms ordinary comments into structured, navigable documentation. This appendix covers everything you need to master `go doc`: how to query documentation from the terminal, how to write comments that `go doc` understands, and how to produce high‑quality, maintainable documentation for your packages.

---

### 1. The Philosophy: Documentation as Code

Go’s documentation system is deliberately simple. There is no separate markup language (no Javadoc, no Sphinx, no Doxygen). You write plain text in comments, and `go doc` formats it for you. The design choices are:

- **No magic tags** – Use `//` comments directly above declarations.
- **No duplication** – The documentation lives next to the code it describes, reducing drift.
- **No build step** – `go doc` reads source files directly; there is no extra generation command.
- **In‑terminal first** – You can get documentation without leaving your editor or shell.

This “less is more” philosophy ensures that writing docs feels as natural as writing code.

---

### 2. Using `go doc` – Querying Documentation

The `go doc` command is your primary tool for exploring package APIs. It works offline, respects your `GOPATH`/modules, and outputs plain text or HTML (via `go doc -html`).

#### Basic Usage

```bash
# Documentation for a package in the standard library
go doc fmt
go doc fmt.Println

# Documentation for a local package (from anywhere inside the module)
go doc ./my/pkg
go doc mypackage.MyFunction

# Show documentation for a symbol including unexported ones
go doc -u fmt.newPrinter

# Show the full source of a function (implementation)
go doc -src fmt.Println

# Show all documentation for a package, including unexported constants/vars
go doc -all fmt
```

**Key flags:**

| Flag | Effect |
|------|--------|
| `-all` | Show all symbols (including unexported and their documentation). |
| `-src` | Print the full source code of the matched symbol. |
| `-u` | Include unexported (private) identifiers. |
| `-c` | Match with case sensitivity (default is case‑insensitive). |
| `-html` | Generate an HTML page (open in browser with `go doc -html fmt > doc.html`). |

#### Navigating the Output

`go doc` shows a package synopsis (the comment before the `package` clause) followed by a **sectioned list** of exported declarations: `CONSTANTS`, `VARIABLES`, `FUNCTIONS`, `TYPES`. For each type, it shows methods and associated functions.

Example output for `go doc strings`:

```
package strings // import "strings"

Package strings implements simple functions to manipulate UTF-8 encoded strings.

const (
    ToLowerSpecial = ...
    ...
)
var NewReader = ...
func Compare(a, b string) int
func Contains(s, substr string) bool
type Builder struct { ... }
    func (b *Builder) Grow(n int)
    ...
```

#### Querying in IDEs

Modern editors (VS Code with the Go extension, Zed, GoLand) integrate `go doc` natively. Hovering over an identifier shows the same documentation. However, mastering the CLI is valuable for scripts and remote debugging.

---

### 3. Writing Documentation for `go doc`

Documentation comments are ordinary line comments (`//`) placed **immediately before** a declaration, with no blank line in between. They become part of the package’s documentation when `go doc` runs.

#### The Package Comment

Every package should have a **package comment** – a comment preceding the `package` clause. It serves as the overview shown by `go doc <package>`.

```go
// Package user provides authentication and profile management for the service.
//
// It supports multiple backends (database, LDAP, OAuth2) and caches
// sessions in memory. This package is concurrency‑safe.
//
// # Example
//
//    u, err := user.New("alice")
//    if err != nil { ... }
//    fmt.Println(u.Name())
package user
```

**Rules:**
- Start with `// Package <name> <description>.` (the dot is required for proper formatting).
- Use blank lines to separate paragraphs.
- Headings are created with `#` (e.g., `# Example`). `go doc` renders them as bold/underline in plain text.
- Indented lines (e.g., with a tab or spaces) are displayed as literal code blocks.

#### Documenting Exported Identifiers

Every exported constant, variable, function, type, and method should have a documentation comment that **begins with the identifier’s name**.

```go
// MaxRetries is the maximum number of attempts before giving up.
const MaxRetries = 3

// Retry calls f repeatedly until it returns nil or the context is cancelled.
// It sleeps with exponential backoff between attempts.
func Retry(ctx context.Context, f func() error) error { ... }

// Server handles HTTP requests.
type Server struct {
    // Addr is the TCP address to listen on, e.g., ":8080".
    Addr string
    // Handler is invoked for each request. If nil, http.DefaultServeMux is used.
    Handler http.Handler
}
```

**Why the name repetition?**
`go doc` strips the identifier name from the first sentence to produce a cleaner synopsis. For example, a comment `// MaxRetries is the maximum...` becomes “the maximum number of attempts” in the package overview. If you omit the name, the synopsis becomes awkward.

#### Linking Between Identifiers

You can create hyperlinks (in HTML output) and plain‑text references by writing `[Name]` or `[pkg.Name]`. For example:

```go
// Parse reads data and returns a [Config] structure.
// See also [filepath.Glob] for file matching.
```

When rendered by `pkg.go.dev` or `go doc -html`, these become clickable links. In terminal output, they are shown as literal `[Config]` – still useful.

#### Deprecation and Bugs

Use the special markers `Deprecated:` and `Bugs:` to signal known issues.

```go
// OldFunc does something the hard way.
//
// Deprecated: Use NewFunc instead, which is faster and thread‑safe.
func OldFunc() { ... }

// Reader reads from a file.
//
// Bugs: The Close method may block if there are pending reads.
type Reader struct { ... }
```

`go doc` will render `Deprecated:` prominently, and tools like `staticcheck` can warn on usage.

#### Inline Code and Preformatted Blocks

- Use backticks for inline code: “The `context` package is required.”
- For multi‑line code examples, indent with a tab:

```go
// Example:
//
//    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
//    defer cancel()
//    err := Retry(ctx, func() error { ... })
```

`go doc` preserves the indentation and displays it as a code block.

---

### 4. Examples – The `example_test.go` Convention

Executable examples (recognized by `go test`) also appear in `go doc` output. Create a file named `example_test.go` in your package:

```go
package user_test

import (
    "fmt"
    "yourmodule/user"
)

func ExampleNew() {
    u, _ := user.New("bob")
    fmt.Println(u.Name())
    // Output: bob
}
```

When you run `go doc user.New`, the example’s output is shown alongside the documentation. Examples are the most reliable way to demonstrate API usage.

**Naming rules:**
- `Example` – shows the whole package.
- `ExampleFunction` – shows a function `Function`.
- `ExampleType_Method` – shows a method of a type.
- Suffix `_output` for a second example that doesn’t run.

---

### 5. Common Mistakes (Even Seasoned Engineers Make)

1. **Blank line between comment and declaration** – `go doc` will ignore the comment.
   ❌
   ```go
   // This comment is lost.

   var Version int
   ```
   ✅
   ```go
   // Version is the current release number.
   var Version int
   ```

2. **Not starting with the identifier name** – produces an ugly synopsis.
   ❌
   ```go
   // Returns the current user.
   func Current() *User
   ```
   Output: `func Current() *User "Returns the current user."` (redundant).
   ✅
   ```go
   // Current returns the logged‑in user.
   func Current() *User
   ```

3. **Forgetting the package comment** – `go doc` will show “package user // import ...”. No overview.

4. **Using `//go:embed` comments or build tags incorrectly** – those directives must be separated from the doc comment by a blank line, else they become part of the doc.
   ```go
   // Package embedtest demonstrates //go:embed.
   //
   //go:embed myfile.txt
   var content string
   ```

5. **Over‑formatting with Markdown** – Only a tiny subset works (headings, code indentation, links). Lists, tables, bold, italics are **not** supported in plain `go doc`. Write plain text; clarity trumps decoration.

---

### 6. Best Practices for Maintainable Documentation

- **Keep it close** – Update the comment whenever you change the declaration’s behavior. No external doc files.
- **Start with the “what”** – The first sentence should be a complete, standalone statement. Many tools (including IDEs) truncate after the first period.
- **Explain the “why” not the “how”** – The code shows the how. The comment explains the contract, side effects, or invariants.
  ```go
  // Process encodes the image and writes it to w.
  // It may allocate up to 10 MB of temporary buffers.
  // The provided context can be used to cancel the operation.
  ```
- **Use examples for complex APIs** – A single `Example` function is worth a paragraph.
- **Document zero values** – If a struct is usable when zero‑valued, say so:
  `// Counter is a concurrency‑safe counter. The zero value is ready to use.`
- **Document concurrency guarantees** – `// This method is safe for concurrent use.` or `// Not goroutine‑safe.`
- **Avoid repeating the type signature** – `// Add adds x and y and returns the result.` is noise. Prefer: `// Add returns the sum of x and y.`

---

### 7. Performance & Tooling Considerations

`go doc` reads .go files directly. There is no indexing, so queries are very fast even on large codebases (milliseconds). However, the `-all` flag can be slow on huge packages because it parses the entire AST.

The documentation comments **do not affect binary size or compilation speed** – they are stripped by the compiler. They only exist in source and in `go doc`’s output.

#### Generating HTML Documentation

Although `go doc -html` works, the canonical way to serve package documentation locally is:

```bash
godoc -http=:6060   # legacy tool, still works but deprecated in favor of pkg.go.dev
```

For modern workflows, use `go run golang.org/x/tools/cmd/godoc@latest -http=:6060`. Alternatively, push to a Git remote and view on `pkg.go.dev/<module>`.

#### Integrating with CI

Lint your documentation using `go vet` and `staticcheck`:

```bash
go vet ./...           # detects malformed doc comments (e.g., missing Package comment)
staticcheck ./...      # checks for deprecated identifiers and misused Deprecated: markers
```

Write a CI step that fails if any exported symbol has no comment:

```bash
# Find exported identifiers without a doc comment (naive but effective)
go list -f '{{.Dir}}' ./... | xargs -I{} bash -c "cd {} && go doc -all . | grep -q '^[A-Z]' || (echo 'Missing doc comment' && exit 1)"
```

---

### 8. Examples: Putting It All Together

Consider a small package `ratelimit`. Here is how you would document it for `go doc`.

**File: `ratelimit.go`**

```go
// Package ratelimit provides token‑bucket rate limiting.
//
// It implements a thread‑safe, memory‑efficient limiter that can be shared
// across many goroutines.
//
// # Example
//
//    lim := ratelimit.New(100, time.Second) // 100 tokens per second
//    if lim.Allow() {
//        // handle request
//    }
package ratelimit

import (
    "context"
    "time"
)

// Limiter controls the rate of events.
type Limiter struct {
    // tokens is the current number of available tokens.
    tokens int
    // rate is how many tokens are added per second.
    rate int
}

// New creates a Limiter that allows up to n events per duration d.
// The zero Limiter is not usable; always use New.
func New(n int, d time.Duration) *Limiter {
    return &Limiter{tokens: n, rate: int(float64(n) / d.Seconds())}
}

// Allow returns true if a token can be taken immediately.
// It never blocks. If false is returned, the caller should wait or drop the event.
func (l *Limiter) Allow() bool {
    // (implementation omitted)
    return true
}

// Wait blocks until a token is available or the context is cancelled.
// It returns ctx.Err() if the context expires.
//
// Deprecated: Use WaitWithPriority instead.
func (l *Limiter) Wait(ctx context.Context) error {
    // ...
    return nil
}
```

**File: `example_test.go`**

```go
package ratelimit_test

import (
    "fmt"
    "time"
    "yourmodule/ratelimit"
)

func ExampleNew() {
    lim := ratelimit.New(5, time.Second)
    for i := 0; i < 10; i++ {
        if lim.Allow() {
            fmt.Println("allowed")
        } else {
            fmt.Println("rejected")
        }
    }
    // Output: allowed (5 times), then rejected (5 times).
}
```

Running `go doc -all ratelimit` would produce:

```
package ratelimit // import "yourmodule/ratelimit"

Package ratelimit provides token‑bucket rate limiting.

It implements a thread‑safe, memory‑efficient limiter that can be shared
across many goroutines.

Example

   lim := ratelimit.New(100, time.Second)
   if lim.Allow() { ... }

type Limiter struct{ ... }
    func New(n int, d time.Duration) *Limiter
    func (l *Limiter) Allow() bool
    func (l *Limiter) Wait(ctx context.Context) error
        Deprecated: Use WaitWithPriority instead.
```

---

### 9. Summary & Advanced Exercises

`go doc` is a minimal yet powerful system that encourages **self‑documenting code** with **zero friction**. By mastering its conventions, you ensure that your packages are usable, discoverable, and professional.

**Key takeaways:**
- Every exported symbol needs a comment starting with its name.
- Package comments are the public face of your library.
- Use examples (`example_test.go`) to show real usage.
- Deprecation and bug markers are recognized by the tooling.
- `go doc` works offline, inside editors, and in CI.

#### Exercises for the Seasoned Engineer

1. **Automated doc coverage**
   Write a `go:generate` directive that runs a custom script to list all exported identifiers missing a doc comment. Integrate it into your CI pipeline to reject pull requests with undocumented API.

2. **Cross‑package linking in a monorepo**
   In a multi‑module workspace (using `go work`), document a function that returns a type from a sibling module. Use `[othermodule.Type]` links and verify that `go doc -html` generates correct relative URLs. Explore the `GODOC` environment variable to point to a local `pkg.go.dev` instance.

3. **Godoc for generated code**
   Use `stringer` or `mockgen` to generate code that lacks documentation. Write a post‑generation step that copies doc comments from the source definitions to the generated output. Measure the impact on `go doc` readability.

4. **Custom output format**
   Write a small CLI tool that consumes `go doc -json` (available from `golang.org/x/tools/cmd/godoc` with `-json` flag) and renders documentation as a static Markdown site. Compare the effort against using `go doc -html`.

By internalizing these practices, you elevate your Go code from “works” to “works and is a joy to consume.” Documentation is not a chore—it is the final expression of a well‑engineered package.
