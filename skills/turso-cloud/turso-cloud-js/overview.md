# JavaScript / TypeScript

## Remote-only — `@tursodatabase/serverless` (recommended)

The recommended TypeScript/JavaScript SDK for Turso Cloud. Uses only the `fetch` API with zero native dependencies — works in Node.js, serverless and edge functions, and any JavaScript runtime.

```bash
npm i @tursodatabase/serverless
```

```javascript
import { connect } from "@tursodatabase/serverless";

const conn = connect({
  url: process.env.TURSO_DATABASE_URL,
  authToken: process.env.TURSO_AUTH_TOKEN,
});

// Execute a query
const result = await conn.prepare("SELECT * FROM users WHERE active = ?").all([1]);
console.log(result);

// Positional parameters
await conn.prepare("SELECT * FROM users WHERE id = ?").get([1]);

// Named parameters
await conn.prepare("INSERT INTO users VALUES (:name)").run({ name: "Alice" });

// Batch multiple statements in an implicit transaction
await conn.batch([
  "CREATE TABLE IF NOT EXISTS todos (id INTEGER PRIMARY KEY, title TEXT, done INTEGER DEFAULT 0)",
  "INSERT INTO todos (title) VALUES ('Buy milk')",
  "INSERT INTO todos (title) VALUES ('Write code')",
]);

// Interactive transaction
const tx = await conn.transaction("write");
await tx.prepare("UPDATE accounts SET balance = balance - ? WHERE id = ?").run([100, 1]);
await tx.prepare("UPDATE accounts SET balance = balance + ? WHERE id = ?").run([100, 2]);
await tx.commit();
```

## Local-first with sync — `@tursodatabase/sync`

For apps that need fast local reads, offline support, or embedded replicas that sync with Turso Cloud. All reads and writes happen against a local database file; sync is explicit (`push()`/`pull()`) and conflict resolution is last-push-wins.

```bash
npm i @tursodatabase/sync
```

```javascript
import { connect } from "@tursodatabase/sync";

const db = await connect({
  path: "local.db",
  url: process.env.TURSO_DATABASE_URL,
  authToken: process.env.TURSO_AUTH_TOKEN,
});

// Pull remote changes to the local database
await db.pull();

// Reads and writes are local
await db.prepare("INSERT INTO todos (title) VALUES (?)").run("Buy milk");

// Push local writes to Turso Cloud
await db.push();
```

> **Warning:** A synced database file must **never** be opened outside the sync SDK (CLI, plain SQLite, or non-sync SDKs will corrupt sync state).

## Package quick reference

| Use case | Package | Install |
|----------|---------|---------|
| Remote-only (Node.js/serverless/edge/backend) | `@tursodatabase/serverless` | `npm i @tursodatabase/serverless` |
| Local-first + sync (Node.js) | `@tursodatabase/sync` | `npm i @tursodatabase/sync` |
| Local-first + sync (Browser/WASM) | `@tursodatabase/sync-wasm` | `npm i @tursodatabase/sync-wasm` |
| Local-first + sync (React Native) | `@tursodatabase/sync-react-native` | `npm i @tursodatabase/sync-react-native` |

## Docs

| Topic | URL |
|-------|-----|
| TypeScript SDK quickstart | `https://docs.turso.tech/sdk/ts/quickstart.md` |
| TypeScript SDK reference | `https://docs.turso.tech/sdk/ts/reference.md` |
| Sync usage | `https://docs.turso.tech/sync/usage.md` |
| Sync conflict resolution | `https://docs.turso.tech/sync/conflict-resolution.md` |
