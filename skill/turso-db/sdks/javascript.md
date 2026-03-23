# JavaScript SDK

Package: `@tursodatabase/database`

## Installation

```bash
npm i @tursodatabase/database
```

For browser/WASM usage, see `sdks/wasm.md` instead.

## Quick Start

```javascript
import { connect } from '@tursodatabase/database';

const db = await connect('mydata.db');
const row = await db.prepare('SELECT 1 AS value').get();
console.log(row); // { value: 1 }
```

## API Reference

### `await connect(path)` → Database

Opens a database connection. Creates the file if it doesn't exist.

```javascript
// File-based database
const db = await connect('mydata.db');

// In-memory database
const db = await connect(':memory:');
```

### class Database

#### `db.prepare(sql)` → Statement

Prepare a SQL statement for execution. This is synchronous.

```javascript
const stmt = db.prepare('SELECT * FROM users WHERE id = ?');
```

#### `await db.exec(sql)`

Execute a SQL statement directly (no results returned). Async.

```javascript
await db.exec('CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)');
```

#### `await db.close()`

Close the database connection. Async.

#### `db.transaction(fn)` → wrapped function

Returns a function that executes the given async function in a transaction.

```javascript
const transfer = db.transaction(async (from, to, amount) => {
    await db.prepare('UPDATE accounts SET balance = balance - ? WHERE id = ?').run(amount, from);
    await db.prepare('UPDATE accounts SET balance = balance + ? WHERE id = ?').run(amount, to);
});
await transfer(1, 2, 100);
```

The returned function has `.deferred`, `.immediate`, and `.exclusive` variants.

#### `await db.pragma(source)` → rows

Execute a PRAGMA statement and return results.

```javascript
const result = await db.pragma('journal_mode');
```

### class Statement

All Statement execution methods are **async** and must be awaited.

#### `await stmt.run([...params])` → info

Execute and return info object with `changes` (modified row count) and `lastInsertRowid`.

```javascript
const info = await db.prepare('INSERT INTO users (name) VALUES (?)').run('Alice');
console.log(info.changes);        // 1
console.log(info.lastInsertRowid); // 1
```

#### `await stmt.get([...params])` → row

Execute and return the first row as an object.

```javascript
const user = await db.prepare('SELECT * FROM users WHERE id = ?').get(1);
console.log(user); // { id: 1, name: 'Alice' }
```

#### `await stmt.all([...params])` → array of rows

Execute and return all rows as an array.

```javascript
const users = await db.prepare('SELECT * FROM users').all();
console.log(users); // [{ id: 1, name: 'Alice' }, ...]
```

#### `for await...of stmt.iterate([...params])` → async iterator

Execute and return an async iterator over rows.

```javascript
for await (const row of db.prepare('SELECT * FROM users').iterate()) {
    console.log(row.name);
}
```

#### `stmt.raw()`, `stmt.pluck()`, `stmt.safeIntegers()` → Statement

Chainable modifiers (synchronous, return `this`).

```javascript
const names = await db.prepare('SELECT name FROM users').pluck().all();
// ['Alice', 'Bob', ...]
```

## Complete Example

```javascript
import { connect } from '@tursodatabase/database';

const db = await connect('app.db');

await db.exec(`
    CREATE TABLE IF NOT EXISTS todos (
        id INTEGER PRIMARY KEY,
        title TEXT NOT NULL,
        done INTEGER DEFAULT 0
    )
`);

// Insert
await db.prepare('INSERT INTO todos (title) VALUES (?)').run('Buy groceries');
await db.prepare('INSERT INTO todos (title) VALUES (?)').run('Write code');

// Query
const pending = await db.prepare('SELECT * FROM todos WHERE done = ?').all(0);
console.log(pending);

// Update
await db.prepare('UPDATE todos SET done = 1 WHERE id = ?').run(1);

await db.close();
```

## Notes

- API is the async variant of `better-sqlite3` — all execution methods (`run`, `get`, `all`, `iterate`, `exec`, `close`) are async and must be awaited
- Install canary releases with `npm i @tursodatabase/database@next` for preview/experimental features
- `backup()`, `serialize()`, `function()`, `aggregate()` are not yet supported
