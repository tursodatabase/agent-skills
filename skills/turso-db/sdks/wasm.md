# WASM (WebAssembly) SDK

Package: `@tursodatabase/database-wasm`

For running Turso in browsers and edge runtimes via WebAssembly.

## Installation

```bash
npm install @tursodatabase/database-wasm
```

## Quick Start

```javascript
import { connect } from '@tursodatabase/database-wasm';

// In-memory database
const db = await connect(':memory:');

// File-based database (uses OPFS)
const db = await connect('my-database.db');

await db.exec('CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT, email TEXT)');

const insert = db.prepare('INSERT INTO users (name, email) VALUES (?, ?)');
await insert.run('Alice', 'alice@example.com');

const users = await db.prepare('SELECT * FROM users').all();
console.log(users);
```

## API Reference

The API mirrors the Node.js SDK (`@tursodatabase/database`):

### `connect(path)` → Database

```javascript
const db = await connect(':memory:');       // In-memory
const db = await connect('my-database.db'); // File via OPFS
```

### Database Methods

#### `db.exec(sql)`

Execute SQL directly (no results).

```javascript
await db.exec('CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)');
```

#### `db.prepare(sql)` → Statement

Prepare a statement for execution.

```javascript
const stmt = db.prepare('SELECT * FROM users WHERE id = ?');
```

#### `db.transaction(fn)`

Execute a function within a transaction.

```javascript
await db.transaction(async () => {
    await db.run('INSERT INTO users (name) VALUES (?)', ['Alice']);
    await db.run('INSERT INTO users (name) VALUES (?)', ['Bob']);
});
```

#### `db.close()`

Close the database connection.

### Statement Methods

#### `stmt.run([...params])` → info

Execute and return `{ changes, lastInsertRowid }`.

```javascript
const info = await db.prepare('INSERT INTO users (name) VALUES (?)').run('Alice');
console.log(info.changes); // 1
```

#### `stmt.get([...params])` → row

Return the first row.

```javascript
const user = await db.prepare('SELECT * FROM users WHERE id = ?').get(1);
```

#### `stmt.all([...params])` → array of rows

Return all rows.

```javascript
const users = await db.prepare('SELECT * FROM users').all();
```

## Browser Sync (Remote Replication)

The sync variant `@tursodatabase/sync-wasm` adds bidirectional sync with Turso Cloud:

```bash
npm install @tursodatabase/sync-wasm
```

```javascript
import { connect } from '@tursodatabase/sync-wasm';

const db = await connect('mydb.db', {
    url: 'libsql://your-db.turso.io',
    authToken: 'your-auth-token',
});

// Pull remote changes to local
await db.pull();

// Make local changes
await db.run('INSERT INTO users (name) VALUES (?)', ['Alice']);

// Push local changes to remote
await db.push();
```

## Required Browser Headers

SharedArrayBuffer is required for WASM threading. Your server must set these headers:

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

### Vite Configuration

If you use Vite, just add the headers to `vite.config.ts` — no extra server setup, proxies, or middleware needed for development:

```javascript
// vite.config.ts
import { defineConfig } from "vite";

export default defineConfig({
  server: {
    headers: {
      "Cross-Origin-Opener-Policy": "same-origin",
      "Cross-Origin-Embedder-Policy": "require-corp",
    },
  },
});
```

This is all you need — Vite's dev server will serve the headers automatically.

### Vercel Configuration

```json
{
    "headers": [
        {
            "source": "/(.*)",
            "headers": [
                { "key": "Cross-Origin-Opener-Policy", "value": "same-origin" },
                { "key": "Cross-Origin-Embedder-Policy", "value": "require-corp" }
            ]
        }
    ]
}
```

## How It Works

- File-based databases use OPFS (Origin Private File System) for browser storage
- Uses a dedicated worker thread for OPFS access
- Shared WebAssembly.Memory for cross-thread communication
- Main thread handles async file I/O, worker thread handles sync operations

## Package Variants

| Package | Purpose |
|---------|---------|
| `@tursodatabase/database-wasm` | Browser WASM (local only) |
| `@tursodatabase/sync-wasm` | Browser WASM with Turso Cloud sync |
| `@tursodatabase/database` | Node.js native bindings |

## Bundler Setup

**Vite is the recommended bundler** for browser projects using Turso WASM.

Use the `/vite` subpath import for Vite projects — it handles WASM module and worker loading correctly in Vite's dev server (works around known Vite issues with WASM + workers):

```javascript
// For local-only database
import { connect } from '@tursodatabase/database-wasm/vite';

// For sync-enabled database
import { connect } from '@tursodatabase/sync-wasm/vite';
```

In production builds, the `/vite` import resolves to the default entry point. In development, it uses a special workaround that inlines the WASM binary to avoid module loading issues.

### All Entry Points

| Environment | database-wasm | sync-wasm |
|-------------|---------------|-----------|
| Default | `@tursodatabase/database-wasm` | `@tursodatabase/sync-wasm` |
| Vite | `@tursodatabase/database-wasm/vite` | `@tursodatabase/sync-wasm/vite` |
| Turbopack | `@tursodatabase/database-wasm/turbopack` | `@tursodatabase/sync-wasm/turbopack` |

## Notes

- Requires browser with SharedArrayBuffer support
- COOP/COEP headers are mandatory — without them, SharedArrayBuffer is unavailable
- OPFS is only available in secure contexts (HTTPS or localhost)
- The WASM package targets `wasm32-wasip1-threads`
- Install canary releases with `npm i @tursodatabase/database-wasm@next` or `npm i @tursodatabase/sync-wasm@next` for preview/experimental features
