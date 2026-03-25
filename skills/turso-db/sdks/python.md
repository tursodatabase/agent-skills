# Python SDK

Package: `pyturso`

**Status:** BETA

## Installation

```bash
pip install pyturso
# or
uv pip install pyturso
```

Requires Python 3.9+.

## Synchronous API (DB-API 2.0)

```python
import turso

conn = turso.connect("my.db")
cur = conn.cursor()

cur.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)")
cur.execute("INSERT INTO users (name) VALUES (?)", ("Alice",))
conn.commit()

cur.execute("SELECT * FROM users")
rows = cur.fetchall()
print(rows)  # [(1, 'Alice')]

conn.close()
```

### Key Methods

| Method | Description |
|--------|-------------|
| `turso.connect(path)` | Open database connection |
| `conn.cursor()` | Create a cursor |
| `cur.execute(sql, params)` | Execute parameterized query |
| `cur.executescript(sql)` | Execute multiple statements |
| `cur.fetchone()` | Fetch single row as tuple |
| `cur.fetchall()` | Fetch all rows as list of tuples |
| `conn.commit()` | Commit current transaction |
| `conn.close()` | Close connection |

## Asynchronous API

```python
import turso.aio

async def main():
    async with turso.aio.connect("my.db") as conn:
        cur = conn.cursor()
        await cur.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)")
        await cur.execute("INSERT INTO users (name) VALUES (?)", ("Alice",))

        await cur.execute("SELECT * FROM users")
        rows = await cur.fetchall()
        print(rows)
```

### Async Methods

| Method | Description |
|--------|-------------|
| `turso.aio.connect(path)` | Async connection (context manager) |
| `await cur.execute(sql, params)` | Execute async query |
| `await cur.executescript(sql)` | Execute multiple statements async |
| `await cur.fetchone()` | Fetch single row async |
| `await cur.fetchall()` | Fetch all rows async |

## Remote Sync

```python
import turso.sync

conn = turso.sync.connect(
    path="local.db",
    remote_url="https://your-db.turso.io",
    auth_token="your-token"
)

# Pull remote changes
conn.pull()

# Make local changes
cur = conn.cursor()
cur.execute("INSERT INTO users (name) VALUES (?)", ("Alice",))
conn.commit()

# Push to remote
conn.push()

# Get sync stats
stats = conn.stats()
print(stats.network_received_bytes)

# Compact local WAL
conn.checkpoint()
```

### Async Remote Sync

```python
import turso.aio.sync

async def main():
    conn = await turso.aio.sync.connect(
        path="local.db",
        remote_url="https://your-db.turso.io",
        auth_token="your-token"
    )
    await conn.pull()
    # ... use database ...
    await conn.push()
```

## Complete Example

```python
import turso

conn = turso.connect("app.db")
cur = conn.cursor()

cur.executescript("""
    CREATE TABLE IF NOT EXISTS todos (
        id INTEGER PRIMARY KEY,
        title TEXT NOT NULL,
        done INTEGER DEFAULT 0
    );
""")

# Insert
cur.execute("INSERT INTO todos (title) VALUES (?)", ("Buy groceries",))
cur.execute("INSERT INTO todos (title) VALUES (?)", ("Write code",))
conn.commit()

# Query
cur.execute("SELECT * FROM todos WHERE done = ?", (0,))
for row in cur.fetchall():
    print(f"id={row[0]}, title={row[1]}")

# Update
cur.execute("UPDATE todos SET done = 1 WHERE id = ?", (1,))
conn.commit()

conn.close()
```

## Notes

- Follows Python DB-API 2.0 (PEP 249) specification
- Parameters use `?` placeholders
- Use context managers (`async with`) for safe resource cleanup in async code
- Built with Maturin/PyO3 (Rust Python bindings)
