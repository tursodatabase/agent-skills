# Go SDK

Module: `turso.tech/database/tursogo`

## Installation

```bash
go get turso.tech/database/tursogo
```

Requires Go 1.24.0+. **No CGO required** — uses purego for Rust FFI.

## Quick Start

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    _ "turso.tech/database/tursogo"
)

func main() {
    db, err := sql.Open("turso", "my.db")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    if _, err := db.Exec("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)"); err != nil {
        log.Fatal(err)
    }
    if _, err := db.Exec("INSERT INTO users (name) VALUES (?)", "Alice"); err != nil {
        log.Fatal(err)
    }

    rows, err := db.Query("SELECT id, name FROM users")
    if err != nil {
        log.Fatal(err)
    }
    defer rows.Close()
    for rows.Next() {
        var id int
        var name string
        if err := rows.Scan(&id, &name); err != nil {
            log.Fatal(err)
        }
        fmt.Printf("id=%d, name=%s\n", id, name)
    }
}
```

## API Reference (database/sql)

Standard Go `database/sql` interface — import the driver with blank identifier:

```go
import _ "turso.tech/database/tursogo"
```

### Opening a Database

```go
db, err := sql.Open("turso", "path/to/db.db")   // File database
db, err := sql.Open("turso", ":memory:")          // In-memory
```

### Execute Statements

```go
result, err := db.Exec("INSERT INTO users (name) VALUES (?)", "Alice")
rowsAffected, _ := result.RowsAffected()
lastID, _ := result.LastInsertId()
```

### Query Rows

```go
rows, err := db.Query("SELECT id, name FROM users WHERE id > ?", 0)
defer rows.Close()
for rows.Next() {
    var id int
    var name string
    err := rows.Scan(&id, &name)
    // use id, name
}
```

### Prepared Statements

```go
stmt, err := db.Prepare("INSERT INTO users (name) VALUES (?)")
defer stmt.Close()
stmt.Exec("Alice")
stmt.Exec("Bob")
```

### Context-Aware Operations

```go
ctx := context.Background()
db.ExecContext(ctx, "INSERT INTO users (name) VALUES (?)", "Alice")
db.QueryContext(ctx, "SELECT * FROM users")
```

## Remote Sync

```go
import "turso.tech/database/tursogo"

ctx := context.Background()

db, err := tursogo.NewTursoSyncDb(ctx, tursogo.TursoSyncConfig{
    Path:      "local.db",          // or ":memory:"
    RemoteUrl: "https://your-db.turso.io",
    AuthToken: "your-token",
})
if err != nil {
    panic(err)
}

conn, err := db.Connect(ctx)
if err != nil {
    panic(err)
}

// Pull remote changes
pulled, err := db.Pull(ctx)  // returns bool

// Make local changes
conn.ExecContext(ctx, "INSERT INTO users (name) VALUES (?)", "Alice")

// Push to remote
err = db.Push(ctx)

// Get sync stats
stats, err := db.Stats(ctx)
fmt.Println(stats.NetworkReceivedBytes)

// Compact local WAL
err = db.Checkpoint(ctx)
```

## Complete Example

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    _ "turso.tech/database/tursogo"
)

func main() {
    db, err := sql.Open("turso", "app.db")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    if _, err := db.Exec(`CREATE TABLE IF NOT EXISTS todos (
        id INTEGER PRIMARY KEY,
        title TEXT NOT NULL,
        done INTEGER DEFAULT 0
    )`); err != nil {
        log.Fatal(err)
    }

    if _, err := db.Exec("INSERT INTO todos (title) VALUES (?)", "Buy groceries"); err != nil {
        log.Fatal(err)
    }
    if _, err := db.Exec("INSERT INTO todos (title) VALUES (?)", "Write code"); err != nil {
        log.Fatal(err)
    }

    rows, err := db.Query("SELECT id, title, done FROM todos WHERE done = ?", 0)
    if err != nil {
        log.Fatal(err)
    }
    defer rows.Close()
    for rows.Next() {
        var id, done int
        var title string
        if err := rows.Scan(&id, &title, &done); err != nil {
            log.Fatal(err)
        }
        fmt.Printf("[%d] %s (done=%d)\n", id, title, done)
    }

    if _, err := db.Exec("UPDATE todos SET done = 1 WHERE id = ?", 1); err != nil {
        log.Fatal(err)
    }
}
```

## Notes

- Uses standard `database/sql` interface — familiar to all Go developers
- No CGO dependency — uses `github.com/ebitengine/purego` for Rust FFI
- Blank import (`_ "turso.tech/database/tursogo"`) registers the `"turso"` driver
- Parameters use `?` placeholders
