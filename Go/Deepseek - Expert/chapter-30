## Chapter 30: Databases & Persistence

Go’s standard library provides `database/sql`, a lightweight, driver-agnostic interface for relational databases. It handles connection pooling, query execution, and transaction management. Rather than building an ORM into the language, Go gives you a minimal, composable foundation—leaving higher-level abstractions to the ecosystem. This chapter covers SQLite and PostgreSQL, but the patterns apply to any SQL database.

---

### 1. Basic Usage

Open a database with `sql.Open`. The first argument is the driver name; the second is a data source name (DSN) that varies by driver. Two popular pure-Go drivers are `modernc.org/sqlite` (CGo-free SQLite) and `github.com/jackc/pgx/v5/stdlib` (PostgreSQL).

```go
import (
	"database/sql"
	"fmt"
	"log"
	"time"

	_ "modernc.org/sqlite"            // registers driver
	_ "github.com/jackc/pgx/v5/stdlib" // registers "pgx" driver
)

func main() {
	// SQLite (in-memory, for development/testing)
	db, err := sql.Open("sqlite", ":memory:")
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	// PostgreSQL (production)
	dsn := "postgres://user:pass@localhost:5432/mydb?sslmode=disable"
	db, err = sql.Open("pgx", dsn)
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	// Verify connectivity with context deadline
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()
	if err := db.PingContext(ctx); err != nil {
		log.Fatalf("ping failed: %v", err)
	}
	fmt.Println("connected")
}
```

The import of a driver package with `_` triggers its `init()` function, which calls `sql.Register` to make the driver available. You never interact with the driver package directly.

**Querying** returns `*sql.Rows`, which must be closed to release the connection back to the pool.

```go
rows, err := db.QueryContext(ctx, "SELECT id, name FROM users WHERE active = $1", true)
if err != nil {
	return err
}
defer rows.Close()

for rows.Next() {
	var id int
	var name string
	if err := rows.Scan(&id, &name); err != nil {
		return err
	}
	fmt.Printf("id=%d name=%s\n", id, name)
}
if err := rows.Err(); err != nil {
	return err // iteration error
}
```

**Single-row queries** use `QueryRowContext`, which returns `*sql.Row`. Call `Scan` directly; `Err` is deferred to `Scan`.

```go
var name string
err := db.QueryRowContext(ctx, "SELECT name FROM users WHERE id = $1", 42).Scan(&name)
if err == sql.ErrNoRows {
	// no result — normal control flow
} else if err != nil {
	return err
}
```

**Exec** for INSERT, UPDATE, DELETE:

```go
res, err := db.ExecContext(ctx, "UPDATE users SET active = false WHERE last_login < $1", cutoff)
if err != nil {
	return err
}
n, _ := res.RowsAffected()
fmt.Println("deactivated", n, "users")
```

**Transactions** group operations atomically:

```go
tx, err := db.BeginTx(ctx, nil)
if err != nil {
	return err
}
defer tx.Rollback() // no-op after Commit

_, err = tx.ExecContext(ctx, "INSERT INTO audit (action) VALUES ($1)", "user_deactivated")
if err != nil {
	return err
}
_, err = tx.ExecContext(ctx, "UPDATE users SET active = false WHERE id = $1", userID)
if err != nil {
	return err
}
return tx.Commit()
```

The `defer tx.Rollback()` pattern ensures cleanup even on panic; after a successful `Commit`, `Rollback` is a no-op.

---

### 2. Under the Hood

The `database/sql` package is a thin orchestration layer. It knows nothing about SQL dialects, wire protocols, or result formats. All that is delegated to a driver that implements a set of interfaces.

**Connection Pool**
`sql.DB` maintains a pool of `*sql.conn` objects, each wrapping a driver’s `driver.Conn`. The pool is goroutine‑safe and manages the lifecycle:

- **Free list** of idle connections (bounded by `SetMaxIdleConns`).
- **Open limit** (bounded by `SetMaxOpenConns`); if exceeded, `QueryContext` blocks until a connection is freed or the context is cancelled.
- **Connection max lifetime** (`SetConnMaxLifetime`) and **idle time** (`SetConnMaxIdleTime`) — connections older or idle longer are closed and replaced, preventing stale TCP sessions from accumulating.

When `db.QueryContext(ctx, query, args...)` is called:

1. The pool picks an idle connection, or opens a new one (up to `MaxOpenConns`).
2. If no connection is available, the caller waits on a channel. The `ctx` is tied to the internal wait, so cancellation aborts the wait and returns an error.
3. Once a connection is obtained, the driver’s `Query` method is invoked.
4. After the query completes (or if `rows.Close()` is called), the connection is returned to the free list.

**Driver Interface**
A driver must implement `driver.Driver` (with `Open`), `driver.Conn`, `driver.Stmt` (prepared statements), `driver.Rows`, `driver.Tx`, and optional interfaces for `driver.Connector`, `driver.Pinger`, `driver.SessionResetter` (to test connection health on reuse), and `driver.NamedValueChecker`.

The connection is responsible for translating `database/sql`’s neutral placeholder syntax (`?` or `$1`) into driver‑specific SQL. For example, `pgx` uses PostgreSQL’s native placeholders `$1`, `$2`, while `sqlite3` uses `?`. The driver may also handle context cancellation by issuing a TCP `CancelRequest` (PostgreSQL) or closing the connection entirely.

**Prepared Statements**
`database/sql` can prepare statements explicitly (`db.Prepare`), but this ties the statement to a single connection, defeating pooling. Instead, most production code relies on the driver’s implicit prepare cache: when you call `db.QueryContext` with a parameterized query, the driver can prepare and cache it internally (pgx does this with a configurable LRU cache). This is more efficient and pool‑friendly.

**Transaction Isolation**
`BeginTx` accepts `*sql.TxOptions` with `Isolation` and `ReadOnly` fields. The driver translates these to database‑specific isolation levels (e.g., `sql.LevelSerializable` maps to `SET TRANSACTION ISOLATION LEVEL SERIALIZABLE`). Not all drivers support all levels.

**Connection Reset**
When a connection is taken from the pool, `database/sql` may call the `SessionResetter` interface (if implemented) to ensure the session is clean: no leftover temporary tables, transaction state, or altered session variables. This is critical for correctness with pooled connections.

---

### 3. Why This Design?

The Go team faced a choice: build an ORM, a query builder, or a minimal interface. They chose the last—a design that mirrors Go’s philosophy of **simplicity, explicitness, and composition**.

- **Minimal interface:** `database/sql` defines only a few interfaces; drivers do the heavy lifting. This keeps the standard library small and avoids blessing one database.
- **Pluggable drivers:** Registering drivers at import time is a simple, zero‑magic mechanism. It avoids a global configuration file or runtime dependency injection.
- **No ORM:** ORMs abstract away SQL, but at a cost: unpredictable query generation, performance cliffs, and a steep learning curve when the abstraction leaks. Go leaves ORMs to third‑party packages, and the ecosystem has converged on lighter alternatives like `sqlc` and query builders.
- **Connection pooling built‑in:** Almost every server application needs a pool. Baking it into `sql.DB` ensures consistent behavior, avoids multiple competing pool implementations, and integrates directly with context cancellation.
- **Explicit error handling:** SQL operations return errors, including `sql.ErrNoRows`. There are no exceptions, no hidden retries, no magic `NULL`‑to‑zero‑value mappings. You see exactly what happens.
- **Context first:** Every public API accepts a context, enabling timeouts, cancellation, and trace propagation without global state. This is essential for microservices and HTTP handlers.

The result is a library that feels “anti‑magic.” You write SQL, you scan results, and you handle errors—exactly what a seasoned engineer expects.

---

### 4. Competing Approaches

**Java: JDBC**
JDBC is conceptually similar to `database/sql`: a driver‑based abstraction with `Connection`, `Statement`, `ResultSet`. However, JDBC is far more verbose, requiring try‑with‑resources blocks, checked exceptions (`SQLException`), and manual resource management. Connection pooling is not built in; you add libraries like HikariCP. Go’s built‑in pool and `defer` drastically reduce boilerplate.

**Python: DB‑API and SQLAlchemy**
Python’s DB‑API 2.0 (PEP 249) offers a cursor‑based interface similar to `database/sql`. However, the ecosystem heavily favors ORMs—SQLAlchemy and Django ORM. While SQLAlchemy’s Core provides a thin query‑builder, the ORM layer maps classes to tables, often leading to N+1 queries and performance surprises. Go’s culture deliberately rejects this “magic” in favor of raw SQL or compile‑time code generation (sqlc).

**Rust: sqlx and Diesel**
Rust’s `sqlx` is close in spirit to Go’s `database/sql` with an async runtime. It’s type‑safe at compile time through macros that verify queries against a live database. Diesel is a full ORM/query‑builder with compile‑time schema checking. Both require a runtime connection pool (like `sqlx::PgPool`). Go’s `database/sql` provides the pool out of the box, but lacks compile‑time query checking—that’s filled by tools like `sqlc`.

**JavaScript/Node.js**
Node.js database drivers are callback or promise‑based, lacking a standardized interface. Each driver (pg, mysql2) has its own API. Connection pooling exists in `pg‑pool` but isn’t universal. Go’s uniform interface is a significant architectural advantage: swapping databases requires changing only the driver import and DSN.

In summary, Go’s design is not revolutionary—it’s deliberately familiar to anyone who’s used a low‑level database driver. The difference is the **ergonomics**: built‑in pooling, context integration, and a cultural preference for explicit SQL.

---

### 5. Common Mistakes

Even experienced engineers fall into these traps:

**1. Forgetting to close `*sql.Rows`**
If you neglect `rows.Close()`, the underlying connection stays busy and never returns to the pool. Eventually the pool starves, and all new queries hang.

```go
// BUG: if an error occurs before rows.Close(), connection leaks
rows, err := db.QueryContext(ctx, "SELECT ...")
if err != nil {
    return err
}
// defer rows.Close() must come immediately after error check
```

Always `defer rows.Close()` right after the error check.

**2. Scanning into wrong‑type or mismatched columns**
`Scan` uses reflection; a mismatch panics or returns an error at runtime. If your query returns 3 columns but you scan only 2, `Scan` returns an error. Always verify the number and types of destination pointers. For nullable columns, use `sql.NullString`, `sql.NullInt64`, or a pointer type (`*string`).

**3. Ignoring `rows.Err()`**
After a `for rows.Next()` loop, you must check `rows.Err()`. The loop may exit prematurely due to a network error, and this error is hidden unless you check.

**4. Connection pool exhaustion**
Default `MaxOpenConns` is unlimited (0). On a high‑traffic server, this can overwhelm the database. Always set `SetMaxOpenConns`, `SetMaxIdleConns`, and `SetConnMaxLifetime` based on your workload. A good starting point: `MaxOpenConns` = 25–100 for PostgreSQL, `MaxIdleConns` = 10–25% of `MaxOpenConns`.

**5. Not using context timeouts**
Every query should have a deadline. Without it, a slow query blocks a goroutine forever, leaking memory and holding a connection.

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
rows, err := db.QueryContext(ctx, "...")
```

**6. Using `sql.DB` as a request‑scoped object**
`sql.DB` is designed to be a long‑lived singleton (one per database). Creating it per HTTP request initializes a new pool each time, wasting resources. Use dependency injection to share a single `*sql.DB` across your application.

**7. Misusing transactions**
- Not calling `Rollback` on error paths: always `defer tx.Rollback()` right after `BeginTx`.
- Holding transactions open too long: keep transactional work short to avoid locking and idle‑in‑transaction states.
- Using `*sql.Tx` concurrently: transactions are not goroutine‑safe.

**8. Relying on ORM‑generated SQL**
Even with a lightweight ORM like GORM, developers sometimes trust the generated SQL without inspection, leading to N+1 queries, missing indexes, or full‑table scans. Always log queries during development and run `EXPLAIN`.

**9. Forgetting that `QueryRow.Scan` defers errors**
`db.QueryRowContext(...).Scan(...)` will not return an error until `Scan` is called. If the query fails, `Scan` returns the error. It’s idiomatic to chain the `.Scan` call.

**10. Mixed‑case column names and scanning**
`Scan` matches destination fields by position, not name. If you later alter column order, scans break silently. Use `sqlx` or `sqlc` to scan into structs by name, or write explicit column lists (never `SELECT *` in production).

---

### 6. Performance Considerations

Database interaction is typically the bottleneck in web services. Go’s side of the equation can be optimized significantly.

**Connection Pool Sizing**
The pool size depends on the database’s ability to handle concurrent connections and your workload concurrency. PostgreSQL works best with `MaxOpenConns` ≤ `4 * number_of_CPU_cores`, or around 25–100 for typical cloud instances. Set `MaxIdleConns` to at least 10 to avoid constant connection churn. Use `db.Stats()` to monitor:

```go
stats := db.Stats()
fmt.Printf("Open=%d InUse=%d Idle=%d WaitCount=%d WaitDuration=%v\n",
    stats.OpenConnections, stats.InUse, stats.Idle,
    stats.WaitCount, stats.WaitDuration)
```

If `WaitCount` grows rapidly, you need more capacity or you’re holding connections too long.

**Prepared Statement Caching**
`database/sql`’s explicit `Prepare` ties a statement to one connection. For high‑throughput, let the driver cache implicitly. `pgx` does this automatically up to a configured capacity (default 512). Use `pgxpool` for even more control over the pool and statement cache.

**Batch Operations**
Inserting rows one‑by‑one in a loop generates a round‑trip per insert. Use `COPY` (PostgreSQL) or multi‑row `INSERT`:

```go
// PostgreSQL multi-row insert with pgx
tx, _ := db.Begin(ctx)
stmt := "INSERT INTO items (name, price) VALUES ($1, $2), ($3, $4), ($5, $6)"
_, err := tx.ExecContext(ctx, stmt, "a", 1, "b", 2, "c", 3)
```

For thousands of rows, `pgx.CopyFrom` is significantly faster, bypassing SQL parsing per row.

**Avoid `interface{}` Scanning**
`Scan` into concrete types. Scanning into `any` or `interface{}` causes allocations and reflection overhead. If you need dynamic results, use `sqlx` or known column sets.

**Use Binary Protocol (pgx)**
`database/sql` with `pgx` uses the text‑based PostgreSQL protocol by default. The pure `pgx` interface (`pgxpool`, `pgx.Conn`) can use the binary protocol, which avoids parsing text representations and is faster for numeric and timestamp types. If performance is critical, consider using `pgx` directly or the `pgx/v5/pgxpool` package.

**Connection Lifetimes**
Databases, load balancers, and firewalls drop idle connections. Set `SetConnMaxLifetime` (e.g., 30 minutes) and `SetConnMaxIdleTime` (e.g., 5 minutes) to recycle connections before the infrastructure kills them. This prevents connection errors at query time.

**Memory Allocations**
Every row scanned allocates destination variables and a new `Rows` structure. Reuse structs where possible in high‑frequency loops (scan into a pre‑allocated slice of structs). Pre‑allocate slices with known capacity when reading large result sets.

**Driver‑level Optimizations**
- `pgx` supports `pgx.Conn.PgConn().ExecParams` with binary format.
- `modernc.org/sqlite` is a pure‑Go SQLite implementation with excellent performance, often within 10–20% of C SQLite.

---

### 7. Best Practices

**1. One `*sql.DB` per database, shared globally**
Create `*sql.DB` in `main` and inject it into your service structs. Never create it inside a request handler.

**2. Repository Pattern**
Define an interface that hides the database. This allows swapping implementations for testing (e.g., an in‑memory mock or a separate test database).

```go
type UserRepository interface {
    GetByID(ctx context.Context, id int) (User, error)
    Create(ctx context.Context, u User) (int, error)
}
type PostgresUserRepo struct {
    db *sql.DB
}
func NewPostgresUserRepo(db *sql.DB) *PostgresUserRepo {
    return &PostgresUserRepo{db: db}
}
```

**3. Use `context` everywhere**
Every database method should accept a `context.Context` as its first argument. This propagates HTTP request deadlines and cancellation signals all the way to the database driver.

**4. Lean on `sqlc` or `sqlx` for type safety**
`sqlc` generates type‑safe Go code from SQL queries at compile time. It eliminates the boilerplate of scanning and validates SQL syntax. `sqlx` provides struct scanning by name and is a lighter runtime alternative. Prefer code generation (`sqlc`) for production services where query correctness is paramount.

**5. Schema Migrations**
Use `golang-migrate/migrate` or `pressly/goose` to manage schema versions. Run migrations at startup or as an init container. Never apply ad‑hoc DDL from application code.

**6. Handle `sql.ErrNoRows` explicitly**
This error means “zero rows,” not “database error.” Use it as a sentinel to return `404 Not Found` or a custom `ErrNotFound` from your repository.

```go
if err == sql.ErrNoRows {
    return User{}, ErrNotFound
}
```

**7. Log Queries (Development Only)**
Use driver wrappers or middleware to log queries and their execution time. In production, use tracing spans instead of plain text logs.

**8. Test with Real Databases**
Unit test repositories against a real database instance—SQLite `:memory:` for fast unit tests, or Dockerized PostgreSQL via testcontainers for integration tests. This catches dialect differences and schema mismatches early.

**9. Set Pool Limits from Configuration**
Expose `MaxOpenConns`, `MaxIdleConns`, `MaxLifetime` as environment variables or config file settings. Tune them under load; start with `MaxOpenConns=25`, `MaxIdleConns=10`, `MaxLifetime=30m`.

**10. Prefer Raw SQL Over ORMs**
Write SQL. It’s explicit, optimizable, and reviewable. ORMs like GORM can be useful for rapid prototyping, but they incur runtime reflection and often generate suboptimal queries. When using GORM, enable `Logger` to see generated SQL and use `.Raw()` for complex queries.

---

### 8. Examples

**Example 1: SQLite CRUD with Repository**

```go
package main

import (
	"context"
	"database/sql"
	"fmt"
	"log"
	"time"

	_ "modernc.org/sqlite"
)

type User struct {
	ID   int
	Name string
}

type UserRepo struct {
	db *sql.DB
}

func NewUserRepo(db *sql.DB) *UserRepo {
	return &UserRepo{db: db}
}

func (r *UserRepo) Create(ctx context.Context, name string) (int, error) {
	res, err := r.db.ExecContext(ctx, "INSERT INTO users (name) VALUES (?)", name)
	if err != nil {
		return 0, err
	}
	id, _ := res.LastInsertId()
	return int(id), nil
}

func (r *UserRepo) ByID(ctx context.Context, id int) (User, error) {
	row := r.db.QueryRowContext(ctx, "SELECT id, name FROM users WHERE id = ?", id)
	var u User
	err := row.Scan(&u.ID, &u.Name)
	if err == sql.ErrNoRows {
		return User{}, fmt.Errorf("user %d: %w", id, ErrNotFound)
	}
	return u, err
}

func (r *UserRepo) Update(ctx context.Context, u User) error {
	_, err := r.db.ExecContext(ctx, "UPDATE users SET name = ? WHERE id = ?", u.Name, u.ID)
	return err
}

func (r *UserRepo) Delete(ctx context.Context, id int) error {
	_, err := r.db.ExecContext(ctx, "DELETE FROM users WHERE id = ?", id)
	return err
}

var ErrNotFound = sql.ErrNoRows // or a custom sentinel

func main() {
	db, err := sql.Open("sqlite", ":memory:")
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	// Create schema
	_, err = db.Exec(`CREATE TABLE users (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT)`)
	if err != nil {
		log.Fatal(err)
	}

	repo := NewUserRepo(db)
	ctx := context.Background()

	id, err := repo.Create(ctx, "Alice")
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("created user id", id)

	u, err := repo.ByID(ctx, id)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("retrieved: %+v\n", u)
}
```

**Example 2: PostgreSQL Transaction with Context Deadline**

```go
import (
	"context"
	"database/sql"
	"fmt"
	"log"
	"time"

	_ "github.com/jackc/pgx/v5/stdlib"
)

type TransferService struct {
	db *sql.DB
}

func (s *TransferService) Transfer(ctx context.Context, from, to int, amount float64) error {
	// Enforce a deadline at the application layer
	ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	tx, err := s.db.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelSerializable})
	if err != nil {
		return fmt.Errorf("begin tx: %w", err)
	}
	defer tx.Rollback()

	// deduct
	_, err = tx.ExecContext(ctx, "UPDATE accounts SET balance = balance - $1 WHERE id = $2", amount, from)
	if err != nil {
		return fmt.Errorf("deduct: %w", err)
	}
	// add
	_, err = tx.ExecContext(ctx, "UPDATE accounts SET balance = balance + $1 WHERE id = $2", amount, to)
	if err != nil {
		return fmt.Errorf("add: %w", err)
	}
	return tx.Commit()
}
```

**Example 3: Repository Interface for Testability**

```go
type AccountRepo interface {
	Balance(ctx context.Context, accountID int) (float64, error)
	UpdateBalance(ctx context.Context, accountID int, amount float64) error
}

// PostgresAccountRepo implements AccountRepo
type PostgresAccountRepo struct {
	db *sql.DB
}

// NewPostgresAccountRepo constructor...

// In-memory mock for testing:
type MockAccountRepo struct {
	Balances map[int]float64
}

func (m *MockAccountRepo) Balance(ctx context.Context, accountID int) (float64, error) {
	if b, ok := m.Balances[accountID]; ok {
		return b, nil
	}
	return 0, ErrNotFound
}

func (m *MockAccountRepo) UpdateBalance(ctx context.Context, accountID int, amount float64) error {
	m.Balances[accountID] = amount
	return nil
}
```

---

### 9. Summary & Exercises

**Summary**
`database/sql` is a minimal yet complete database access layer. It provides connection pooling, context propagation, and an extensible driver model. The Go community prefers explicit SQL and thin abstractions over heavy ORMs, leading to predictable performance and fewer hidden bugs. By combining `database/sql` with code generation (`sqlc`), structured logging, and the repository pattern, you can build robust, observable persistence layers.

**Exercises**

1. **Build a Transactional User Repository with Audit Trail**
   Create a `UserRepository` interface with methods `Create`, `GetByID`, and `Deactivate`. The `Deactivate` method must execute within a transaction: insert an audit log entry and update the user’s `active` flag. If the audit insert fails, the user update must not be applied. Write a PostgreSQL implementation using `database/sql`, and include a test with a real database (either SQLite `:memory:` or a testcontainer). Ensure your test covers the case where the audit insert fails.

2. **Connection Pool Monitor**
   Write a function `MonitorPool(db *sql.DB, interval time.Duration)` that periodically calls `db.Stats()` and logs the number of open, idle, in‑use connections, and the cumulative wait count/duration. If the wait count increases by more than `X` within a minute, log a warning. Integrate this monitor into a simple HTTP server that queries a database and observe the logs under load (use `go-wrk` or `hey`). Use the `slog` package for structured logging.

3. **Database‑Backed Cache with Write‑Through**
   Design a cache layer that fronts a `UserRepository`. The cache must be thread‑safe and support Get, Put, and Delete operations with write‑through semantics: every `Put` and `Delete` must update the database synchronously, then invalidate the cache entry. On `Get`, check the cache first; on a miss, fetch from the database and populate the cache. Use a `sync.Mutex` or `sync.RWMutex` to protect the cache map. Implement the cache as a concrete type that satisfies the `UserRepository` interface, wrapping a real repository. Then write a benchmark to compare the latency of cached vs. uncached reads.
