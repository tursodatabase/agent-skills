# Go

| Use case | Package | Install |
|----------|---------|---------|
| Local-first + sync | `tursogo` | `go get turso.tech/database/tursogo` |
| Remote-only (libsql protocol) | `libsql-client-go` | `go get github.com/tursodatabase/libsql-client-go` |

For most applications, a local database with sync gives faster reads, offline support, and lower latency. Remote-only access is useful when you cannot store a local database file (e.g., stateless serverless environments).

## Local-first with sync — `tursogo`

All reads and writes happen against a local database file; sync is explicit (`Push()`/`Pull()`) and conflict resolution is last-push-wins. No CGO required.

```go
package main

import (
    "context"
    "os"

    "turso.tech/database/tursogo"
)

func main() {
    ctx := context.Background()

    db, err := tursogo.NewTursoSyncDb(ctx, tursogo.TursoSyncConfig{
        Path:      "local.db",
        RemoteUrl: os.Getenv("TURSO_DATABASE_URL"),
        AuthToken: os.Getenv("TURSO_AUTH_TOKEN"),
    })
    if err != nil {
        panic(err)
    }

    conn, err := db.Connect(ctx)
    if err != nil {
        panic(err)
    }

    // Pull remote changes to the local database
    if _, err := db.Pull(ctx); err != nil {
        panic(err)
    }

    // Reads and writes are local
    if _, err := conn.ExecContext(ctx, "INSERT INTO users (name) VALUES (?)", "Alice"); err != nil {
        panic(err)
    }

    // Push local writes to Turso Cloud
    if err := db.Push(ctx); err != nil {
        panic(err)
    }
}
```

> **Warning:** A synced database file must **never** be opened outside the sync SDK (CLI, plain SQLite, or non-sync SDKs will corrupt sync state).

## Remote-only — `libsql-client-go`

Connects directly to the remote database over the libsql protocol through the standard `database/sql` interface — no local file needed.

```go
package main

import (
    "database/sql"
    "fmt"
    "os"

    _ "github.com/tursodatabase/libsql-client-go/libsql"
)

func main() {
    url := os.Getenv("TURSO_DATABASE_URL") + "?authToken=" + os.Getenv("TURSO_AUTH_TOKEN")

    db, err := sql.Open("libsql", url)
    if err != nil {
        panic(err)
    }
    defer db.Close()

    rows, err := db.Query("SELECT id, name FROM users")
    if err != nil {
        panic(err)
    }
    defer rows.Close()
    for rows.Next() {
        var id int
        var name string
        if err := rows.Scan(&id, &name); err != nil {
            panic(err)
        }
        fmt.Printf("User: %d %s\n", id, name)
    }
}
```

## Docs

| Topic | URL |
|-------|-----|
| Go SDK quickstart | `https://docs.turso.tech/sdk/go/quickstart.md` |
| Sync usage | `https://docs.turso.tech/sync/usage.md` |
