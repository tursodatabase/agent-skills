# Python

| Use case | Package | Install |
|----------|---------|---------|
| Local-first + sync | `pyturso` | `pip install pyturso` |
| Remote-only (libsql protocol) | `libsql` | `pip install libsql` |

For most applications, a local database with sync gives faster reads, offline support, and lower latency. Remote-only access is useful when you cannot store a local database file (e.g., stateless serverless environments).

## Local-first with sync — `pyturso`

All reads and writes happen against a local database file; sync is explicit (`push()`/`pull()`) and conflict resolution is last-push-wins. The API follows the standard Python `sqlite3` / DB-API 2.0 interface.

```python
import os
import turso.sync

db = turso.sync.connect(
    "app.db",
    remote_url=os.environ["TURSO_DATABASE_URL"],
    auth_token=os.environ["TURSO_AUTH_TOKEN"],
)

# Reads and writes are local
db.execute("INSERT INTO users (name) VALUES (?)", ("Bob",))
db.commit()

# Push local writes to Turso Cloud
db.push()

# Pull remote changes to the local database
db.pull()
```

> **Warning:** A synced database file must **never** be opened outside the sync SDK (CLI, plain SQLite, or non-sync SDKs will corrupt sync state).

## Remote-only — `libsql`

Connects directly to the remote database over the libsql protocol — no local file needed.

```python
import os
import libsql

conn = libsql.connect(
    database=os.environ["TURSO_DATABASE_URL"],
    auth_token=os.environ["TURSO_AUTH_TOKEN"],
)

conn.execute("CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, name TEXT)")
conn.execute("INSERT INTO users (name) VALUES (?)", ("Alice",))
conn.commit()

rows = conn.execute("SELECT * FROM users").fetchall()
print(rows)
```

## Docs

| Topic | URL |
|-------|-----|
| Python SDK quickstart | `https://docs.turso.tech/sdk/python/quickstart.md` |
| Sync usage | `https://docs.turso.tech/sync/usage.md` |
