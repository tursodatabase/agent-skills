# React Native SDK

Package: `@tursodatabase/sync-react-native`

React Native bindings for Turso with bidirectional sync to Turso Cloud.

## Installation

```bash
npm install @tursodatabase/sync-react-native
```

### iOS Setup

```bash
cd ios && pod install
```

### Android Setup

Ensure `minSdkVersion` is 21+ in `android/build.gradle`.

## Quick Start

```typescript
import { Database, getDbPath } from '@tursodatabase/sync-react-native';

// Get platform-specific path
const dbPath = getDbPath('myapp.db');

// Create database with sync
const db = new Database({
    path: dbPath,
    url: 'libsql://your-db.turso.io',
    authToken: 'your-auth-token',
});

// Connect (bootstraps if first sync)
await db.connect();

// Query (fast local reads)
const users = await db.all('SELECT * FROM users');

// Write locally
await db.run('INSERT INTO users (name) VALUES (?)', ['Alice']);

// Sync with remote
await db.push();  // Push local changes
await db.pull();  // Pull remote changes

await db.close();
```

## API Reference

### Constructor

```typescript
const db = new Database({
    path: string,           // Local database path (use getDbPath())
    url?: string,           // Turso Cloud URL (libsql://...)
    authToken?: string,     // Authentication token
    remoteEncryption?: {    // Optional encryption config
        cipher: string,
        key: string,
    },
});
```

If `url` and `authToken` are omitted, the database is local-only (no sync).

### Database Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `connect()` | `Promise<void>` | Open/bootstrap the database |
| `exec(sql)` | `Promise<void>` | Execute SQL without results |
| `run(sql, params?)` | `Promise<{ changes, lastInsertRowid }>` | Execute with result info |
| `get(sql, params?)` | `Promise<row>` | Query single row |
| `all(sql, params?)` | `Promise<row[]>` | Query all rows |
| `prepare(sql)` | `Statement` | Create prepared statement |
| `transaction(fn)` | `Promise<void>` | Execute within transaction |
| `push()` | `Promise<void>` | Push local changes to remote |
| `pull()` | `Promise<void>` | Pull remote changes to local |
| `sync()` | `Promise<void>` | Push then pull (bidirectional) |
| `stats()` | `Promise<Stats>` | Get sync statistics |
| `checkpoint()` | `Promise<void>` | Compact local WAL |
| `close()` | `Promise<void>` | Close database |

### Platform Paths

```typescript
import { getDbPath, paths } from '@tursodatabase/sync-react-native';

// Recommended: auto-selects correct platform directory
const dbPath = getDbPath('myapp.db');

// Available path directories
paths.documents  // iOS: Documents, Android: database dir
paths.database   // iOS: Documents, Android: database dir
paths.files      // iOS: Documents, Android: files dir
paths.library    // iOS: Library, Android: files dir
```

## Sync Operations

```typescript
const db = new Database({
    path: getDbPath('replica.db'),
    url: 'libsql://your-db.turso.io',
    authToken: 'your-token',
});

await db.connect();

// Pull remote changes to local replica
await db.pull();

// Make local changes
await db.run('INSERT INTO users (name) VALUES (?)', ['Alice']);

// Push local changes to remote
await db.push();

// Or sync both directions at once
await db.sync();

// Check sync stats
const stats = await db.stats();
console.log(stats);
```

## Encryption

```typescript
const db = new Database({
    path: getDbPath('encrypted.db'),
    url: 'libsql://your-db.turso.io',
    authToken: 'your-token',
    remoteEncryption: {
        cipher: 'aes256gcm',  // or 'aes128gcm', 'chacha20poly1305', 'aegis256', etc.
        key: 'base64-encoded-key',
    },
});
```

### Supported Ciphers

| Cipher | Description |
|--------|-------------|
| `aes256gcm` | AES-GCM 256-bit |
| `aes128gcm` | AES-GCM 128-bit |
| `chacha20poly1305` | ChaCha20-Poly1305 |
| `aegis256` | AEGIS-256 |
| `aegis256x2` | AEGIS-256-X2 |
| `aegis256x4` | AEGIS-256-X4 |
| `aegis128l` | AEGIS-128L |
| `aegis128x2` | AEGIS-128-X2 |
| `aegis128x4` | AEGIS-128-X4 |

## Experimental: Partial Sync

For large databases, sync only a subset of data:

```typescript
const db = new Database({
    path: getDbPath('replica.db'),
    url: 'libsql://your-db.turso.io',
    authToken: 'your-token',
    partialSyncExperimental: {
        bootstrapStrategy: {
            kind: 'prefix',
            length: 100,
        },
        segmentSize: 4096,
        prefetch: true,
    },
});
```

## Complete Example

```typescript
import React, { useEffect, useState } from 'react';
import { View, Text, FlatList } from 'react-native';
import { Database, getDbPath } from '@tursodatabase/sync-react-native';

export default function App() {
    const [users, setUsers] = useState([]);

    useEffect(() => {
        async function init() {
            const db = new Database({
                path: getDbPath('app.db'),
                url: 'libsql://your-db.turso.io',
                authToken: 'your-token',
            });

            await db.connect();
            await db.pull();

            await db.exec(`
                CREATE TABLE IF NOT EXISTS users (
                    id INTEGER PRIMARY KEY,
                    name TEXT NOT NULL
                )
            `);

            await db.run('INSERT INTO users (name) VALUES (?)', ['Alice']);
            await db.push();

            const rows = await db.all('SELECT * FROM users');
            setUsers(rows);
        }
        init();
    }, []);

    return (
        <FlatList
            data={users}
            keyExtractor={(item) => String(item.id)}
            renderItem={({ item }) => <Text>{item.name}</Text>}
        />
    );
}
```

## Notes

- Uses JSI (JavaScript Interface) for direct native access — no bridge overhead
- Local reads are fast (no network) — sync happens explicitly via push/pull
- Native libraries: `.dylib` (iOS), `.so` (Android)
- CocoaPods for iOS, CMake + JNI for Android
