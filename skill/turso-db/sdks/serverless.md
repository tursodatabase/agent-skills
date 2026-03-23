# Serverless SDK

Package: `@tursodatabase/serverless`

For connecting to Turso Cloud from serverless and edge functions (Cloudflare Workers, Vercel, Deno Deploy, etc.). Uses only `fetch()` — no native bindings or WASM required.

## Installation

```bash
npm i @tursodatabase/serverless
```

## Quick Start

```javascript
import { connect } from "@tursodatabase/serverless";

const conn = connect({
  url: process.env.TURSO_DATABASE_URL,
  authToken: process.env.TURSO_AUTH_TOKEN,
});

const rows = await conn.execute("SELECT * FROM users WHERE active = ?", [1]);
console.log(rows);
```

## API Reference

### `connect(config)` → Connection

Creates a new connection. This is lightweight — no network I/O happens until the first query.

```javascript
const conn = connect({
  url: "https://your-db-turso.turso.io",
  authToken: "your-token",
});
```

### class Connection

#### `await conn.execute(sql, args?)` → result

Execute a SQL statement and return all results.

```javascript
const result = await conn.execute("SELECT * FROM users WHERE id = ?", [123]);
console.log(result.rows);
```

#### `await conn.exec(sql)`

Execute a SQL statement (or multiple statements) directly.

```javascript
await conn.exec("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)");
```

#### `await conn.prepare(sql)` → Statement

Prepare a SQL statement. **This is async** (unlike the native SDK) — it fetches column metadata from the server.

```javascript
const stmt = await conn.prepare("SELECT * FROM users WHERE id = ?");
const user = await stmt.get([123]);
```

#### `await conn.batch(statements)`

Execute multiple SQL statements in a batch.

```javascript
await conn.batch([
  "CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, name TEXT)",
  "INSERT INTO users (name) VALUES ('Alice')",
  "INSERT INTO users (name) VALUES ('Bob')",
]);
```

#### `await conn.pragma(pragma)` → rows

Execute a PRAGMA statement.

```javascript
const result = await conn.pragma("journal_mode");
```

#### `conn.transaction(fn)` → wrapped function

Returns an async function that executes `fn` in a transaction. Has `.deferred`, `.immediate`, `.exclusive` variants.

```javascript
const transfer = conn.transaction(async (from, to, amount) => {
  await conn.execute("UPDATE accounts SET balance = balance - ? WHERE id = ?", [amount, from]);
  await conn.execute("UPDATE accounts SET balance = balance + ? WHERE id = ?", [amount, to]);
});
await transfer(1, 2, 100);
```

#### `await conn.close()`

Close the connection and clean up the server-side stream.

### class Statement

All execution methods are **async**.

#### `await stmt.run(args?)` → `{ changes, lastInsertRowid }`

Execute and return info about affected rows.

```javascript
const info = await stmt.run(["Alice"]);
console.log(info.changes);        // 1
console.log(info.lastInsertRowid); // 1
```

#### `await stmt.get(args?)` → row or undefined

Execute and return the first row as an object.

```javascript
const user = await stmt.get([123]);
```

#### `await stmt.all(args?)` → array of rows

Execute and return all rows.

```javascript
const users = await stmt.all([true]);
```

#### `for await...of stmt.iterate(args?)` → async iterator

Stream rows one at a time (memory-efficient for large result sets).

```javascript
for await (const row of stmt.iterate(["electronics"])) {
  console.log(row.id, row.name);
}
```

#### `stmt.raw()`, `stmt.pluck()`, `stmt.safeIntegers()` → Statement

Chainable modifiers (synchronous, return `this`).

```javascript
const ids = await (await conn.prepare("SELECT id FROM users")).pluck().all();
// [1, 2, 3, ...]
```

#### `stmt.columns()` → array

Returns column metadata (available immediately after `prepare`).

```javascript
const stmt = await conn.prepare("SELECT id, name FROM users");
console.log(stmt.columns()); // [{ name: 'id', type: 'INTEGER' }, { name: 'name', type: 'TEXT' }]
```

#### `stmt.reader` → boolean

Whether the statement returns data (`true` for SELECT, `false` for INSERT/UPDATE/DELETE).

## Concurrency

A Connection is **single-stream** — concurrent calls are automatically serialized. For parallel queries, create multiple connections:

```javascript
const config = { url: process.env.TURSO_URL, authToken: process.env.TURSO_TOKEN };

const [users, orders] = await Promise.all([
  connect(config).execute("SELECT * FROM users WHERE active = 1"),
  connect(config).execute("SELECT * FROM orders WHERE status = 'pending'"),
]);
```

## libSQL Compatibility Layer

For migrating from `@libsql/client`, use the compat import:

```javascript
import { createClient } from "@tursodatabase/serverless/compat";

const client = createClient({
  url: process.env.TURSO_DATABASE_URL,
  authToken: process.env.TURSO_AUTH_TOKEN,
});

// Execute
const result = await client.execute("SELECT * FROM users WHERE id = ?", [123]);
console.log(result.rows);
console.log(result.columns);
console.log(result.rowsAffected);

// Batch
await client.batch([
  "CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, name TEXT)",
  "INSERT INTO users (name) VALUES ('Alice')",
]);

// Execute multiple statements
await client.executeMultiple("INSERT INTO t VALUES (1); INSERT INTO t VALUES (2);");

// Close (synchronous)
client.close();
```

**Compat layer limitations**: `transaction()` and `sync()` are not supported.

## Vite + Bun: Loading Environment Variables

Vite does not expose `process.env` to client code by default, and with Bun the standard `define` approach can fail because `process.env.TURSO_*` is undefined at config evaluation time. Use `loadEnv` to explicitly load `.env` files:

```javascript
// vite.config.js
import { defineConfig, loadEnv } from 'vite';

export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '');
  return {
    define: {
      'process.env.TURSO_DATABASE_URL': JSON.stringify(env.TURSO_DATABASE_URL),
      'process.env.TURSO_AUTH_TOKEN': JSON.stringify(env.TURSO_AUTH_TOKEN),
    },
  };
});
```

The third argument `''` to `loadEnv` removes the `VITE_` prefix requirement, so it loads all env vars from `.env` regardless of prefix.

## Key Differences from Native SDK

| Feature | Native (`@tursodatabase/database`) | Serverless (`@tursodatabase/serverless`) |
|---------|------------------------------------|-----------------------------------------|
| `prepare()` | Sync | **Async** (fetches column metadata) |
| Transport | File I/O / OPFS | `fetch()` over HTTP |
| Environment | Node.js, Browser (WASM) | Any runtime with `fetch()` |
| `connect()` | Async (opens file) | Sync (no I/O until first query) |
| Compat layer | Same package | `@tursodatabase/serverless/compat` |
