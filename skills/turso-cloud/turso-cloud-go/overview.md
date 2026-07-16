# Go

| Use case | Package | Install |
|----------|---------|---------|
| Local-first + sync | `tursogo` | `go get turso.tech/database/tursogo` |
| Remote-only (libsql protocol) | `libsql-client-go` | `go get github.com/tursodatabase/libsql-client-go` |

`tursogo` provides a local SQLite-compatible database with explicit sync (`push()`/`pull()`) against Turso Cloud — conflict resolution is last-push-wins, and a synced database file must **never** be opened outside the sync SDK. `libsql-client-go` connects directly to the remote database over the libsql protocol.

## Docs

| Topic | URL |
|-------|-----|
| Go SDK quickstart | `https://docs.turso.tech/sdk/go/quickstart.md` |
| Sync usage | `https://docs.turso.tech/sync/usage.md` |
