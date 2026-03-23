# Sync (Remote Replication)

Turso supports bidirectional sync between a local embedded database and a remote Turso Cloud instance. This enables offline-first applications where reads are always fast (local) and writes sync when connectivity is available.

## CRITICAL: Always Use the Sync SDK

**A synced database file MUST only be opened through the sync SDK** (e.g., `turso::sync::Builder` in Rust, `turso.sync.connect()` in Python, `tursogo.NewTursoSyncDb()` in Go, etc.).

**NEVER open a synced database with:**
- The `tursodb` CLI
- A non-sync SDK (e.g., plain `turso::Builder` / `turso.connect()`)
- SQLite directly (`sqlite3`, `better-sqlite3`, etc.)
- Any other tool that can open SQLite/WAL files

**Why:** The sync engine relies on specific WAL invariants (frame positions, revision tracking, CDC state) to function correctly. Any external access can trigger a checkpoint or modify the WAL in ways that break these invariants, **corrupting the sync state permanently**. The database may appear fine locally but will fail to push/pull, or worse, silently lose data on the next sync.

If you need to inspect a synced database, make a copy of the file first and open the copy.

## You Must Write Your Own Sync Logic

**There is no automatic background sync.** The sync SDK gives you `push()` and `pull()` methods, but you are responsible for calling them at the right time. The SDK will never call them for you.

This is by design — your app knows best when to sync (e.g., on user action, on a timer, when connectivity changes, after a write batch).

## Core Operations

Every SDK exposes these sync operations:

### `pull()`

Downloads remote changes to the local replica. Returns `true` if new changes were applied.

1. Sends current local revision to remote
2. Remote responds with all frames since that revision
3. Frames are applied to local WAL
4. Local metadata updated with new revision

After pull, local queries immediately see remote changes. When `pull()` returns `true`, the app should likely refresh any UI or cached state that depends on database content — the local data has changed and stale reads are a common source of bugs in local-first apps.

### `push()`

Uploads local changes to the remote.

1. Collects logical changes from local CDC table (since last push)
2. Sends changes as batched SQL to remote
3. Remote applies changes atomically
4. Local metadata updated

### `checkpoint()`

Compacts the local WAL by transferring synced frames back to the main database file, then truncating the WAL. Also maintains a revert database that preserves pre-sync page state, enabling the sync engine to roll back local changes when pulling remote updates.

### `stats()`

Returns current sync engine statistics:

| Field | Description |
|-------|-------------|
| `cdcOperations` | Number of pending CDC operations (since last push) |
| `mainWalSize` | Current main WAL file size in bytes |
| `revertWalSize` | Current revert WAL file size in bytes |
| `networkSentBytes` | Total bytes uploaded to remote |
| `networkReceivedBytes` | Total bytes downloaded from remote |
| `lastPullUnixTime` | Unix timestamp of last pull (null if never pulled) |
| `lastPushUnixTime` | Unix timestamp of last push (null if never pushed) |
| `revision` | Current synced revision (opaque token, null if not yet synced) |

## TypeScript Sync Examples

### Setup

```typescript
import { connect } from "@tursodatabase/sync";

const db = await connect({
  path: "local.db",
  url: "libsql://your-db-org.turso.io",
  authToken: "your-token",
  longPollTimeoutMs: 5_000, // recommended — enables long-polling for pull()
});
```

The `connect()` options:

| Option | Description |
|--------|-------------|
| `path` | Local file path for the database (required) |
| `url` | Remote Turso Cloud URL. Omit for local-only. Can be a `() => string \| null` function for deferred sync |
| `authToken` | Auth token string, or `() => Promise<string>` for short-lived credentials |
| `remoteEncryption` | `{ key: string, cipher: string }` for encrypted remote databases |
| `transform` | Callback to transform mutations before push (conflict resolution) |
| `longPollTimeoutMs` | When set, `pull()` holds the connection open until changes arrive or timeout. Max effective value is **5000ms** (server caps it). **Recommended** — set this to `5000` to avoid wasteful polling |
| `tracing` | `'error' \| 'warn' \| 'info' \| 'debug' \| 'trace'` |

### Push After Writes

```typescript
// Write locally, then push to remote
await db.prepare("INSERT INTO todos (title) VALUES (?)").run("Buy milk");
await db.prepare("INSERT INTO todos (title) VALUES (?)").run("Write code");
await db.push();
```

### Long-Polling Pull (Recommended)

With `longPollTimeoutMs` set (max 5s), `pull()` blocks until remote changes arrive or the timeout expires. This is the preferred approach — it gives near-instant change detection without wasteful polling.

```typescript
// Loop: pull() blocks until remote has new changes (or 5s timeout)
async function watchForChanges() {
  while (true) {
    const hasChanges = await db.pull();
    if (hasChanges) {
      console.log("Remote changed — refreshing");
      // IMPORTANT: refresh UI / invalidate queries — local data has changed
    }
  }
}
watchForChanges();
```

### Pull on Interval (Without Long-Polling)

If you cannot use long-polling, poll on a timer instead:

```typescript
setInterval(async () => {
  const hasChanges = await db.pull();
  if (hasChanges) {
    console.log("Got new changes from remote");
  }
}, 30_000);
```

### Separate Push and Pull Loops (Recommended)

Run pull and push in separate loops so they don't block each other — pull can long-poll for changes while push fires independently after writes.

```typescript
// Pull loop — runs continuously, reacts to remote changes fast
async function pullLoop(db) {
  while (true) {
    try {
      const hasChanges = await db.pull();
      if (hasChanges) {
        // IMPORTANT: refresh UI / invalidate queries — local data has changed
      }
    } catch (err) {
      console.error("Pull failed:", err);
      await new Promise(r => setTimeout(r, 1_000)); // back off briefly on error
    }
  }
}

// Push loop — sends local changes to remote on its own cadence
async function pushLoop(db, intervalMs = 5_000) {
  while (true) {
    try {
      await db.push();
    } catch (err) {
      console.error("Push failed:", err);
    }
    await new Promise(r => setTimeout(r, intervalMs));
  }
}

pullLoop(db);
pushLoop(db);
```

If you combine push and pull in a single loop, pull blocks push (especially with long-polling), delaying outbound changes.

### Checkpoint Periodically

```typescript
// Compact the WAL after syncing to keep the local file small
setInterval(async () => {
  await db.checkpoint();
}, 5 * 60_000); // every 5 minutes
```

### Monitor Sync Stats

```typescript
const stats = await db.stats();
console.log(`Pending changes: ${stats.cdcOperations}`);
console.log(`WAL size: ${stats.mainWalSize} bytes`);
console.log(`Last pull: ${stats.lastPullUnixTime}`);
console.log(`Last push: ${stats.lastPushUnixTime}`);
```

## Bootstrap

On first sync with an empty local database, a **bootstrap** downloads the full remote database:

- **Full bootstrap** (default): Downloads all pages — the local replica becomes a complete copy
- **Partial bootstrap** (experimental): Downloads only a subset of data, reducing initial bandwidth. Remaining pages are fetched on demand.

Set `bootstrapIfEmpty: false` to skip the automatic bootstrap on first pull. The database will be created locally but remain empty until you explicitly pull. This is useful when you want the database ready for sync but want to delay the initial download (e.g., waiting for user login or network conditions).

### Partial Sync Configuration

Partial sync is experimental and available in JavaScript, WASM, React Native, Python, and Go SDKs. Configuration options:

| Parameter | Description |
|-----------|-------------|
| `bootstrapStrategy` | How to select initial data: `prefix` (load first N bytes) or `query` (load pages touched by a SQL statement) |
| `segmentSize` | Load pages in batches of this many bytes. E.g., with `segmentSize=131072` (128KB), accessing page 1 loads pages 1–32 together |
| `prefetch` | When `true`, the sync engine proactively fetches pages that are likely to be accessed soon based on access patterns |

## SDK Availability

| SDK | Sync Support | Sync Package/Feature |
|-----|-------------|---------------------|
| Rust | Yes | `turso` crate with `sync` feature |
| Python | Yes | `turso.sync` / `turso.aio.sync` modules |
| Go | Yes | `tursogo.NewTursoSyncDb()` |
| WASM | Yes | `@tursodatabase/sync-wasm` (separate package) |
| React Native | Yes | Built into `@tursodatabase/sync-react-native` |
| JavaScript (Node.js) | Yes | `@tursodatabase/sync` (separate package) |

## Important Notes

- **NEVER open a synced database outside the sync SDK** — CLI, SQLite, or non-sync SDKs will corrupt sync state
- Sync is **explicit** — call push/pull manually; there is no automatic background sync
- Local reads never block on network — they always read from the local replica
- Pull is idempotent — safe to call multiple times
- Both push and pull are atomic — partial failures don't corrupt the database
- Remote encryption is supported via cipher + key configuration (see SDK-specific docs)
- **Security:** Sync uploads local database contents to Turso Cloud. Enable remote encryption (see `references/encryption.md`) if the database contains sensitive data. Ensure auth tokens are kept secret and not hardcoded in source.
