# Chapter 30: Databases & Persistence

## Basic Usage

Interacting with databases in Go starts with the standard library’s `database/sql` package. It provides a generic interface around SQL databases, requiring a driver (e.g., `github.com/lib/pq` for PostgreSQL, `modernc.org/sqlite` for pure‑Go SQLite). Here’s how you connect, query, and modify data.

```go
package main

import (
    "context"
    "database/sql"
    "log"
    "time"

    _ "github.com/lib/pq"          // PostgreSQL driver (registered via init)
    _ "modernc.org/sqlite"         // Pure‑Go SQLite driver
)

type User struct {
    ID        int
    Name      string
    Email     string
    CreatedAt time.Time
}

func main() {
    // Connect to PostgreSQL
    pgDSN := "postgres://user:pass@localhost/dbname?sslmode=disable"
    pgDB, err := sql.Open("postgres", pgDSN)
    if err != nil {
        log.Fatal(err)
    }
    defer pgDB.Close()

    // Verify the connection is alive (optional but recommended)
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    if err := pgDB.PingContext(ctx); err != nil {
        log.Fatal(err)
    }

    // Connect to SQLite (file-based)
    sqliteDB, err := sql.Open("sqlite", "./app.db")
    if err != nil {
        log.Fatal(err)
    }
    defer sqliteDB.Close()

    // Example: Query a single user
    var user User
    row := pgDB.QueryRowContext(ctx,
        "SELECT id, name, email, created_at FROM users WHERE id = $1", 42)
    err = row.Scan(&user.ID, &user.Name, &user.Email, &user.CreatedAt)
    if err == sql.ErrNoRows {
        log.Println("User not found")
    } else if err != nil {
        log.Fatal(err)
    }

    // Example: Insert a user (using RETURNING for auto‑generated ID)
    var newID int
    err = pgDB.QueryRowContext(ctx,
        "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id",
        "Alice", "alice@example.com").Scan(&newID)
    if err != nil {
        log.Fatal(err)
    }

    // Example: Update with named parameters (via sqlx or manual map)
    result, err := pgDB.ExecContext(ctx,
        "UPDATE users SET email = $1 WHERE id = $2",
        "newalice@example.com", newID)
    if err != nil {
        log.Fatal(err)
    }
    rowsAffected, _ := result.RowsAffected()
    log.Printf("Updated %d rows", rowsAffected)
}
```

**Important:** `sql.Open` does not establish a connection – it creates a connection pool. You must call `Ping` (or `PingContext`) to verify the database is reachable. The `_` import of the driver registers it with `database/sql`.

## Under the Hood

### The `database/sql` Abstraction

`database/sql` defines a set of interfaces (`Connector`, `Driver`, `Conn`, `Stmt`, `Tx`, `Rows`) that each driver implements. When you call `sql.Open`, the driver’s `Open` method returns a `driver.Conn`. The `database/sql` package wraps this connection in its own `*sql.DB`, which manages a **connection pool** – a set of zero or more underlying connections that can be reused.

**Connection Pool Behavior:**
- `SetMaxOpenConns(n)`: Maximum number of open connections (default unlimited). Exceeding this makes `Query`/`Exec` block until a connection is free.
- `SetMaxIdleConns(n)`: Maximum number of idle connections kept open (default 2). Higher values reduce connection churn but increase resource usage.
- `SetConnMaxLifetime(d)`: Maximum time a connection can be reused. After that, it’s closed and replaced.
- `SetConnMaxIdleTime(d)`: Maximum time an idle connection stays open before being closed (Go 1.15+).

**Connection Lifecycle:**
1. `sql.DB` acquires an idle connection from the pool or opens a new one (up to `MaxOpenConns`).
2. The connection is used for the operation (`Query`, `Exec`, `BeginTx`).
3. If the operation is a `Query`, the connection is **not returned** until `rows.Close()` is called (explicitly or via `defer rows.Close()`).
4. Otherwise, the connection returns to the idle pool immediately after the call.

A common leak: forgetting `rows.Close()` ties up a connection forever, eventually exhausting the pool.

### Transactions & Savepoints

`database/sql` provides explicit transactions via `db.BeginTx(ctx, &sql.TxOptions{Isolation: sql.LevelSerializable})`. The returned `*sql.Tx` has its own `Exec`, `Query`, `Prepare` methods that run within the transaction.

Under the hood, `BeginTx` acquires a connection, executes `BEGIN` (or equivalent), and returns a wrapper that holds that connection exclusively. All subsequent operations on the `Tx` use the same connection. `Commit` or `Rollback` release the connection back to the pool.

**Savepoints** (nested transactions) are not part of the standard interface, but you can execute `SAVEPOINT x` and `ROLLBACK TO SAVEPOINT x` as raw SQL statements – they are driver‑specific.

### `sql.NullString` and Zero Values

SQL databases distinguish `NULL` from empty string / zero integer. Go’s zero values cannot represent that. The package offers `sql.NullString`, `sql.NullInt64`, `sql.NullTime`, etc. Each has a `Valid` bool. When scanning a `NULL` column, `Valid` becomes `false`. This avoids the trap of interpreting a missing value as the zero value.

```go
var email sql.NullString
err := row.Scan(&email)
if email.Valid {
    fmt.Println(email.String)
} else {
    fmt.Println("email is NULL")
}
```

## Why This Design?

The Go team deliberately avoided an ORM (Object‑Relational Mapper) or a heavy query builder in the standard library. Their philosophy:

1. **SQL is a well‑understood language.** Instead of inventing a Go‑flavored DSL to generate SQL, embrace SQL directly. This makes the code transparent – you see exactly what query runs, no magical translation.

2. **Explicit error handling.** Every database operation returns `error`. No hidden exceptions that unwind the stack. This forces the developer to consider failures at each step – connection loss, constraint violation, deadlock, etc.

3. **Connection pooling is a runtime concern.** Many languages (Python’s `sqlite3`, PHP’s PDO) do not pool connections by default, leading developers to reinvent pooling. Go builds it into the standard library, correctly configured via a few methods.

4. **Context awareness from the start.** Every blocking operation (`QueryContext`, `ExecContext`, `PingContext`) accepts a `context.Context`. This enables request‑scoped timeouts, cancellation, and tracing without third‑party libraries.

5. **No built‑in migration tool.** The team expects you to use external tools (`golang‑migrate`, `goose`, or custom scripts). This keeps `database/sql` focused on runtime data access, not schema management.

## Competing Approaches

### Java (JPA / Hibernate)
- **Philosophy:** Object‑Relational Mapping abstracts away the database. You work with entity objects, and the ORM generates SQL.
- **Trade‑off:** Simplifies CRUD for simple cases. But complex queries require JPQL or native SQL, and the abstraction can leak (N+1 problem, lazy loading exceptions).
- **Go’s stance:** No ORM in stdlib. Use raw SQL or lightweight helpers (`sqlx`, `squirrel`). Accept that mapping rows to structs is manual.

### Python (SQLAlchemy / Django ORM)
- **Philosophy:** Rich query builders, connection pooling in external libraries, and asynchronous drivers (e.g., `asyncpg`).
- **Trade‑off:** Extremely flexible but heavy. Type checking is optional (unless using `mypy`). Go’s static types and explicit `Scan` make the data flow clear.
- **Go’s advantage:** No GIL – concurrent database queries use real parallelism. Goroutines are cheap; Python’s async still runs in one thread.

### Rust (`sqlx` / `diesel`)
- **Philosophy:** Compile‑time checked queries (`sqlx::query!` macro) or a type‑safe ORM (`diesel`).
- **Trade‑off:** Rust’s macro system allows embedding SQL that is validated against a real database schema at compile time. This eliminates a class of runtime errors.
- **Go’s choice:** No macro system, so runtime checks via `Scan` are the norm. However, Go’s `sql.Null*` and `err` handling are simpler to understand for most teams.

### C# (Entity Framework Core)
- **Philosophy:** LINQ‑based querying, migrations, and change tracking.
- **Trade‑off:** Extremely productive for schema‑first or code‑first workflows, but the generated SQL can be surprising. Go prefers explicitness: you write the SQL, you own the performance.

## Common Mistakes

### 1. Not Closing `rows`
```go
rows, err := db.Query("SELECT id FROM users")
// Missing: defer rows.Close()
for rows.Next() {
    // ...
}
// Connection stays open until rows is garbage collected (or forever)
```
**Fix:** Always `defer rows.Close()` immediately after checking the error.

### 2. Ignoring Context Timeouts
```go
resp, err := db.Exec("INSERT INTO logs (payload) VALUES ($1)", hugePayload)
```
If the query takes 10 minutes, your handler might have already timed out, but the database operation continues. **Fix:** Always use `ExecContext`, `QueryContext` and pass a context with a deadline.

### 3. Misusing `sql.Open` – Not Pinging
`sql.Open` does not validate the DSN or reachability. A typo in the hostname will succeed until you call `Ping` or the first query. **Fix:** Call `PingContext` during application startup.

### 4. Connection Leak via Prepared Statements
Prepared statements in `database/sql` live on a connection. If you prepare a statement on a `*sql.DB`, it is cached across connections. But if you prepare on a `*sql.Tx`, the statement is bound to that transaction’s connection. Failing to close it leaks the connection. **Fix:** Use `db.PrepareContext` for global prepared statements, and always call `stmt.Close()`.

### 5. Handling NULL with Zero Values
```go
var age int
row.Scan(&age) // if column is NULL, age becomes 0 (ambiguous with actual age 0)
```
**Fix:** Use `sql.NullInt64` and check `Valid`.

### 6. Not Handling `sql.ErrNoRows`
Unlike many languages where a query returning no rows yields an empty result set, Go returns a specific error. Novices often forget to distinguish `ErrNoRows` from other errors.

```go
err := row.Scan(...)
if err != nil {
    // Wrong: treats "not found" as fatal
    log.Fatal(err)
}
```

### 7. Prepared Statement Cache Bloat
`db.SetMaxIdleConns` high + many different queries = many prepared statements kept open. Each driver has a limit (PostgreSQL: `max_prepared_transactions`). **Fix:** Monitor prepared statement count, or use `db.SetConnMaxLifetime` to rotate connections and free statements.

## Performance Considerations

### Row Scanning Overhead
`rows.Scan` uses reflection to assign values to struct fields. For high‑throughput services, this reflection shows up in CPU profiles. **Mitigations:**
- Use `sqlx` (which still uses reflection but caches mappings).
- Generate per‑query scanning code (e.g., `sqlc` compiles SQL to type‑safe Go).
- Use raw `[]sql.RawBytes` and manual conversion for extreme cases.

### Connection Pool Sizing
The optimal `MaxOpenConns` depends on database capabilities. For PostgreSQL, a good starting point is `(2 * GOMAXPROCS) + (number of expected concurrent requests)`. Too high: database connection overhead, lock contention. Too low: request queueing.

**Rule of thumb:** Set `MaxOpenConns` to `(database max connections - 10)` for other services. Set `MaxIdleConns` equal to `MaxOpenConns` to avoid connection churn under load.

### Prepared Statements vs. Raw SQL
Prepared statements reduce parse overhead for repeated queries, but they consume database resources. For a high‑variance workload (thousands of distinct queries), raw SQL with sanitized arguments may be faster because you avoid the prepare round‑trip.

**Example decision:**
- API endpoint that runs the same `SELECT ... WHERE id = ?` → use `db.PrepareContext` once.
- Bulk import with 100 different `INSERT` patterns → use `ExecContext` with raw SQL.

### Large Result Sets
`rows.Next()` loads rows one at a time into driver‑specific buffers. But if each row contains a large text column (e.g., 10 MB), the memory usage accumulates as you hold the row reference. **Fix:** Process rows in chunks, using `rows.Scan` into a struct and then discarding.

```go
for rows.Next() {
    var hugeText string
    if err := rows.Scan(&hugeText); err != nil { ... }
    // Process hugeText, then let it be garbage collected
}
```

### Batch Inserts
Many drivers support extended protocol for batch inserts. `database/sql` does not expose a standard interface, but you can use `ExecContext` with a multi‑value insert:
```sql
INSERT INTO users (name) VALUES ($1), ($2), ($3)
```
For thousands of rows, use `db.Begin()` and commit every 500–1000 rows to avoid memory explosion in the driver.

### Indexing and Query Planning
Go can’t help you here – you must use `EXPLAIN` in your database. However, Go’s performance profiling (`pprof`) can reveal slow queries by instrumenting your SQL calls with custom spans.

## Best Practices

### 1. Use Contexts Everywhere
Pass a context from the request layer (e.g., HTTP request context) down to `QueryContext`. This ensures that a client disconnect cancels the database operation, freeing server resources.

### 2. Define Repository Interfaces
Don’t pass `*sql.DB` directly throughout your business logic. Abstract behind a small interface:

```go
type UserRepository interface {
    GetByID(ctx context.Context, id int) (*User, error)
    Create(ctx context.Context, user *User) error
}
```

This makes testing with a mock or in‑memory implementation trivial. The concrete repository owns the `*sql.DB` (or `*sql.Tx` for transactional boundaries).

### 3. Handle Transactions Explicitly
Write a helper function that manages `Begin`, `Commit`, `Rollback`:

```go
func WithTransaction(ctx context.Context, db *sql.DB, fn func(*sql.Tx) error) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer func() {
        if p := recover(); p != nil {
            tx.Rollback()
            panic(p)
        } else if err != nil {
            tx.Rollback()
        } else {
            err = tx.Commit()
        }
    }()
    err = fn(tx)
    return err
}
```

### 4. Use `sqlc` or `sqlx` for Production
`database/sql` is low‑level. For real projects:
- **sqlc:** Generates type‑safe, efficient Go code from SQL. Zero runtime reflection.
- **sqlx:** Extends `database/sql` with `StructScan`, `Get`, `Select` – reduces boilerplate but uses reflection.
- **squirrel (or goqu):** Query builders for dynamic SQL (e.g., complex filters).

### 5. Always Set Connection Limits
Even in development, set `SetMaxOpenConns(10)` to catch leaks early. In production, monitor `sql.DB.Stats()`:

```go
type DBStats struct {
    MaxOpenConnections int // Maximum open connections allowed
    OpenConnections    int // Currently open
    InUse              int // Connected and not idle
    Idle               int // Idle connections in pool
    WaitCount          int // Total number of connections waited for
    WaitDuration       time.Duration // Total time blocked waiting for a connection
    MaxIdleClosed      int // Closed because of SetMaxIdleConns
    MaxLifetimeClosed  int // Closed because of SetConnMaxLifetime
}
```

Export these metrics to your observability system.

### 6. Prefer `sql.Null*` or Pointers for Nullable Columns
```go
type User struct {
    Age *int // nil means NULL
}
```
When scanning, use `sql.NullInt64` internally and convert. Pointers are idiomatic in Go for optional values.

## Examples

### Example 1: Repository Pattern with PostgreSQL

```go
type PostgresUserRepo struct {
    db *sql.DB
}

func NewPostgresUserRepo(db *sql.DB) *PostgresUserRepo {
    return &PostgresUserRepo{db: db}
}

func (r *PostgresUserRepo) GetByID(ctx context.Context, id int) (*User, error) {
    query := `SELECT id, name, email, created_at FROM users WHERE id = $1`
    row := r.db.QueryRowContext(ctx, query, id)
    
    var u User
    err := row.Scan(&u.ID, &u.Name, &u.Email, &u.CreatedAt)
    if err == sql.ErrNoRows {
        return nil, nil // not found
    }
    if err != nil {
        return nil, fmt.Errorf("scan user: %w", err)
    }
    return &u, nil
}

func (r *PostgresUserRepo) Create(ctx context.Context, u *User) error {
    query := `INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id`
    err := r.db.QueryRowContext(ctx, query, u.Name, u.Email).Scan(&u.ID)
    if err != nil {
        return fmt.Errorf("insert user: %w", err)
    }
    return nil
}
```

### Example 2: Transactional Update Across Two Tables

```go
func TransferMoney(ctx context.Context, db *sql.DB, fromID, toID int, amount decimal.Decimal) error {
    return WithTransaction(ctx, db, func(tx *sql.Tx) error {
        // Deduct from source
        _, err := tx.ExecContext(ctx,
            "UPDATE accounts SET balance = balance - $1 WHERE id = $2 AND balance >= $1",
            amount, fromID)
        if err != nil {
            return fmt.Errorf("debit: %w", err)
        }
        
        // Add to destination
        _, err = tx.ExecContext(ctx,
            "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
            amount, toID)
        if err != nil {
            return fmt.Errorf("credit: %w", err)
        }
        return nil
    })
}
```

### Example 3: Using `sqlc` Generated Code

Schema:
```sql
CREATE TABLE authors (
  id   BIGSERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  bio  TEXT
);

-- name: GetAuthor :one
SELECT * FROM authors
WHERE id = $1 LIMIT 1;
```

Run `sqlc generate` → produces type‑safe Go:
```go
func (q *Queries) GetAuthor(ctx context.Context, id int64) (Author, error) {
    row := q.db.QueryRowContext(ctx, getAuthor, id)
    var i Author
    err := row.Scan(&i.ID, &i.Name, &i.Bio)
    return i, err
}
```

### Example 4: SQLite In‑Memory for Testing

```go
func TestUserRepo(t *testing.T) {
    db, err := sql.Open("sqlite", ":memory:")
    require.NoError(t, err)
    defer db.Close()
    
    // Create schema
    _, err = db.Exec(`CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)`)
    require.NoError(t, err)
    
    repo := NewPostgresUserRepo(db) // works because SQLite satisfies sql.DB interface
    // ... test
}
```

## Summary & Exercises

**Summary:**  
Go’s `database/sql` package provides a minimal but powerful abstraction over SQL databases. It forces explicit error handling, context propagation, and connection pooling awareness. While it lacks ORM conveniences, this transparency prevents hidden performance issues and makes the code easier to optimize. The key to production success lies in correctly configuring the connection pool, always using contexts, and abstracting repositories behind interfaces. For complex applications, adopt helpers like `sqlx` or code generators like `sqlc` to reduce boilerplate without sacrificing visibility.

**Key Takeaways:**
- `sql.Open` only validates parameters; always `Ping` after.
- Connection pooling is managed by `SetMaxOpenConns`, `SetMaxIdleConns`, etc.
- Use `*Context` methods and pass request‑scoped contexts.
- Handle `sql.ErrNoRows` explicitly – it’s not a fatal error.
- Prefer repository interfaces for testability and clean architecture.
- For high performance, consider `sqlc` or careful scanning logic.

### Exercises

1. **Build a Thread‑Safe Connection Pool Wrapper**  
   Implement a `DBPool` struct that wraps `*sql.DB`. Add a `GetConn(ctx context.Context) (*sql.Conn, error)` method that enforces a maximum connection lifetime and retries with exponential backoff if no connection is available. Include a `Close()` method that waits for all connections to be returned.

2. **Implement Optimistic Locking with Version Field**  
   Create a `Product` struct with fields `ID`, `Name`, `Price`, `Version` (int). Write a `UpdatePrice(ctx context.Context, id int, newPrice decimal.Decimal, currentVersion int) error` function that updates only if the version matches. On success, increment the version. If a version conflict occurs, return a custom error and retry up to 3 times.

3. **Build a Multi‑Database Migrator**  
   Write a CLI tool that reads SQL migration files (e.g., `001_create_users.sql`, `002_add_email_index.sql`) and applies them to either PostgreSQL or SQLite based on a `--driver` flag. Use a `migrations` table to track applied versions. Implement `up` and `down` commands. Ensure migrations run within a transaction so a failure rolls back automatically.
