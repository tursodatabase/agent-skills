# Rust SDK

Crate: `turso`

## Installation

```bash
cargo add turso
cargo add tokio --features full
```

## Quick Start

```rust
use turso::Builder;

#[tokio::main]
async fn main() {
    let db = Builder::new_local("my.db").build().await.unwrap();
    let conn = db.connect().unwrap();
    conn.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)", ()).await.unwrap();
    conn.execute("INSERT INTO users (name) VALUES (?1)", ("Alice",)).await.unwrap();

    let mut rows = conn.query("SELECT * FROM users", ()).await.unwrap();
    while let Some(row) = rows.next().await.unwrap() {
        println!("id={}, name={}", row.get_value(0).unwrap(), row.get_value(1).unwrap());
    }
}
```

## API Reference

### Builder

```rust
// Local file database
let db = Builder::new_local("path/to/db.db").build().await?;

// In-memory database
let db = Builder::new_local(":memory:").build().await?;
```

### Database

```rust
let conn = db.connect()?;
```

### Connection

#### `conn.execute(sql, params)` → Result

Execute INSERT, UPDATE, DELETE, or DDL statements.

```rust
conn.execute("INSERT INTO users (name) VALUES (?1)", ("Alice",)).await?;
conn.execute("UPDATE users SET name = ?1 WHERE id = ?2", ("Bob", 1)).await?;
conn.execute("DELETE FROM users WHERE id = ?1", (1,)).await?;
```

#### `conn.query(sql, params)` → Rows

Query data with streaming results.

```rust
let mut rows = conn.query("SELECT id, name FROM users WHERE id > ?1", (0,)).await?;
while let Some(row) = rows.next().await? {
    let id: i64 = row.get_value(0)?;
    let name: String = row.get_value(1)?;
}
```

#### `conn.prepare(sql)` → Statement

Prepare a statement for repeated execution.

```rust
let stmt = conn.prepare("INSERT INTO users (name) VALUES (?1)").await?;
// Use stmt for multiple executions
```

## Remote Sync (Optional)

Enable the `sync` feature for cloud sync:

```bash
cargo add turso --features sync
```

```rust
use turso::sync::Builder;

let db = Builder::new_remote("local.db", "https://your-db.turso.io", "auth-token")
    .build()
    .await?;

let conn = db.connect()?;

// Pull remote changes
db.pull().await?;

// Make local changes
conn.execute("INSERT INTO users (name) VALUES (?1)", ("Alice",)).await?;

// Push to remote
db.push().await?;

// Get sync stats
let stats = db.stats().await?;
println!("received: {} bytes", stats.network_received_bytes);

// Force WAL checkpoint
db.checkpoint().await?;
```

### Remote Encryption

```rust
use turso::sync::{Builder, RemoteEncryptionCipher};

let db = Builder::new_remote("local.db", "https://your-db.turso.io", "auth-token")
    .encryption_cipher(RemoteEncryptionCipher::Aes256Gcm)
    .encryption_key("your-hex-key")
    .build()
    .await?;
```

## Notes

- All operations are async — requires `tokio` runtime
- Use `()` for no parameters, tuples for positional params: `(?1, ?2)` → `(val1, val2)`
- Results are streamed via `rows.next().await`
