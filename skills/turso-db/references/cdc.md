# Change Data Capture (CDC)

Turso supports CDC for tracking all database changes (inserts, updates, deletes) in real-time per connection.

## Enabling CDC

```sql
PRAGMA capture_data_changes_conn('<mode>[,custom_table_name]');
```

### Modes

| Mode | Description |
|------|-------------|
| `off` | Disable CDC for this connection |
| `id` | Log only the rowid (most compact) |
| `before` | Capture row state before updates/deletes |
| `after` | Capture row state after inserts/updates |
| `full` | Capture both before and after states (recommended for audit trails) |

### Custom CDC Table

By default, changes go to `turso_cdc`. Specify a custom table name:

```sql
PRAGMA capture_data_changes_conn('full,my_audit_log');
```

## CDC Table Schema (v2)

| Column | Type | Description |
|--------|------|-------------|
| `change_id` | INTEGER | Monotonically increasing unique ID (primary key) |
| `change_time` | INTEGER | Unix timestamp (seconds) — not guaranteed to be strictly increasing |
| `change_txn_id` | INTEGER | Transaction ID — groups CDC rows belonging to the same transaction |
| `change_type` | INTEGER | `1` = INSERT, `0` = UPDATE, `-1` = DELETE, `2` = COMMIT |
| `table_name` | TEXT | Affected table name (`"sqlite_schema"` for DDL) |
| `id` | INTEGER | Rowid of affected row |
| `before` | BLOB | Row state before change (NULL for INSERT) |
| `after` | BLOB | Row state after change (NULL for DELETE) |
| `updates` | BLOB | Granular column modifications (for UPDATE) |

COMMIT records (`change_type = 2`) mark transaction boundaries. In autocommit mode, one COMMIT record is emitted per statement. In explicit transactions (`BEGIN...COMMIT`), a single COMMIT record is emitted at the end.

## Complete Example

```sql
-- Enable full CDC
PRAGMA capture_data_changes_conn('full');

-- Create and populate a table
CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    name TEXT
);

INSERT INTO users VALUES (1, 'John'), (2, 'Jane');
UPDATE users SET name = 'John Doe' WHERE id = 1;
DELETE FROM users WHERE id = 2;

-- View all changes
SELECT * FROM turso_cdc;
-- Shows: schema creation, 2 inserts, 1 update, 1 delete
```

## Important Notes

- CDC records are visible even before a transaction commits
- Failed operations (e.g., constraint violations) are NOT recorded
- Changes to the CDC table itself are also logged if CDC is enabled
- In `full` mode, each UPDATE writes 3x data (before + after + actual WAL write)
- Schema changes (ALTER TABLE, DROP TABLE, etc.) appear with `table_name = 'sqlite_schema'`
- If you modify table schema, `table_columns_json_array()` returns the CURRENT schema, not historical. Track schema versions manually before making changes.
