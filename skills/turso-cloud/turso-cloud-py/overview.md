# Python

| Use case | Package | Install |
|----------|---------|---------|
| Local-first + sync | `pyturso` | `pip install pyturso` |
| Remote-only (libsql protocol) | `libsql` | `pip install libsql` |

`pyturso` provides a local SQLite-compatible database with explicit sync (`push()`/`pull()`) against Turso Cloud — conflict resolution is last-push-wins, and a synced database file must **never** be opened outside the sync SDK. `libsql` connects directly to the remote database over the libsql protocol.

## Docs

| Topic | URL |
|-------|-----|
| Python SDK quickstart | `https://docs.turso.tech/sdk/python/quickstart.md` |
| Sync usage | `https://docs.turso.tech/sync/usage.md` |
