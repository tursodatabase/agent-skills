# MVCC (Multi-Version Concurrency Control)

Turso supports MVCC for concurrent read/write transactions with snapshot isolation.

**Status:** Experimental — not production ready.

## Enabling MVCC

```sql
PRAGMA journal_mode = experimental_mvcc;
```

To switch back to WAL:

```sql
PRAGMA journal_mode = wal;
```

Switching modes triggers a checkpoint to persist all pending changes.

## BEGIN CONCURRENT

MVCC enables `BEGIN CONCURRENT` transactions that allow multiple concurrent readers and writers:

```sql
BEGIN CONCURRENT TRANSACTION;
-- Read and write operations...
COMMIT;
```

## How It Works

1. Each concurrent transaction gets a unique ID and begin timestamp
2. Reads see a consistent snapshot as of the begin timestamp
3. No locks are acquired — reads and writes happen without blocking
4. At COMMIT time, conflict detection runs:
   - If another transaction modified the same row after this transaction started → `SQLITE_BUSY` error
   - If an exclusive transaction (BEGIN IMMEDIATE) is active → `SQLITE_BUSY` error
5. On conflict, ROLLBACK and retry

## Transaction Types Comparison

| Type | Syntax | Behavior |
|------|--------|----------|
| Deferred (default) | `BEGIN` | No locks until first read/write |
| Immediate | `BEGIN IMMEDIATE` | Exclusive write lock immediately |
| Concurrent | `BEGIN CONCURRENT` | MVCC snapshot isolation, no locks |

## Conflict Detection

Write-write conflicts occur when:
- The row was modified by another active transaction
- The row was modified by a transaction that committed after this transaction's begin timestamp

## Retry Pattern

```sql
-- Application-level retry loop
BEGIN CONCURRENT;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- If SQLITE_BUSY: ROLLBACK and retry the entire transaction
```

## Interaction with Exclusive Transactions

- Concurrent transactions CAN read and write while an exclusive transaction is active
- Concurrent transactions CANNOT commit while an exclusive transaction holds the lock
- Use `BEGIN IMMEDIATE` only when you need exclusive access (e.g., schema changes)

**Best practice:** For maximum concurrency, use `BEGIN CONCURRENT` for all write transactions.

