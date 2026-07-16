# Rust

| Use case | Package | Install |
|----------|---------|---------|
| Local-first + sync | `turso` (with `sync` feature) | `cargo add turso --features sync` |
| Remote-only (libsql protocol) | `libsql` (with `remote` feature) | `cargo add libsql --features remote` |

For most applications, a local database with sync gives faster reads, offline support, and lower latency. Remote-only access is useful when you cannot store a local database file (e.g., stateless serverless environments). All operations are async — a `tokio` runtime is required.

## Local-first with sync — `turso`

All reads and writes happen against a local database file; sync is explicit (`push()`/`pull()`) and conflict resolution is last-push-wins.

```bash
cargo add turso --features sync
cargo add tokio --features full
```

```rust
use turso::sync::Builder;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let db = Builder::new_remote("app.db")
        .with_remote_url(&std::env::var("TURSO_DATABASE_URL")?)
        .with_auth_token(&std::env::var("TURSO_AUTH_TOKEN")?)
        .build()
        .await?;

    let conn = db.connect().await?;

    // Reads and writes are local
    conn.execute("INSERT INTO users (name) VALUES (?)", ("Bob",)).await?;

    // Push local writes to Turso Cloud
    db.push().await?;

    // Pull remote changes to the local database
    db.pull().await?;

    Ok(())
}
```

> **Warning:** A synced database file must **never** be opened outside the sync SDK (CLI, plain SQLite, or non-sync SDKs will corrupt sync state).

## Remote-only — `libsql`

Connects directly to the remote database over HTTP — no local file needed, no C compiler required.

```bash
cargo add libsql --features remote
cargo add tokio --features full
```

```rust
use libsql::Builder;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let url = std::env::var("TURSO_DATABASE_URL")?;
    let token = std::env::var("TURSO_AUTH_TOKEN")?;

    let db = Builder::new_remote(url, token).build().await?;
    let conn = db.connect()?;

    let mut rows = conn.query("SELECT * FROM users", ()).await?;
    while let Some(row) = rows.next().await? {
        let id: i64 = row.get(0)?;
        let name: String = row.get(1)?;
        println!("User: {} {}", id, name);
    }

    Ok(())
}
```

## Docs

| Topic | URL |
|-------|-----|
| Rust SDK quickstart | `https://docs.turso.tech/sdk/rust/quickstart.md` |
| Sync usage | `https://docs.turso.tech/sync/usage.md` |
