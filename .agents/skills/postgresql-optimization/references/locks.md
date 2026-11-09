# Locks and Concurrency

## Description

Why a query or transaction is blocked. PostgreSQL uses **relation-level** locks (eight modes), **row-level** locks (four modes), and lightweight locks for internal structures. Readers and writers do not block each other for ordinary DML thanks to MVCC, but DDL, explicit locking, and conflicting row updates do. Deadlocks are detected after `deadlock_timeout`; long waits are visible in `pg_locks` and `pg_stat_activity`.

## When to use

- Queries hang with no CPU use.
- DDL or migrations block reads/writes.
- `SELECT ... FOR UPDATE` queues stall.
- Deadlock errors appear in logs or `pg_stat_database.deadlocks` rises.
- A queue-like workload processes the same row twice.
- Autovacuum or `CREATE INDEX` is blocked.

## Checks

```sql
-- Who is waiting, and on what
SELECT pid, now() - query_start AS duration, state,
       wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY duration DESC;

-- Blocked sessions and the pids blocking them (PostgreSQL 9.6+)
SELECT pid,
       pg_blocking_pids(pid) AS blockers,
       wait_event_type,
       wait_event,
       query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;

-- Lock details
SELECT l.pid, l.locktype, l.relation::regclass, l.mode,
       l.granted, l.waitstart, a.query
FROM pg_locks l
JOIN pg_stat_activity a ON a.pid = l.pid
WHERE NOT l.granted
ORDER BY l.waitstart;
```

| Wait type | Meaning |
|---|---|
| `Lock` | Heavyweight lock (relation, tuple, transaction, advisory) |
| `LWLock` | Lightweight internal lock (WAL, buffer mapping) |
| `BufferPin` | Waiting for a buffer pin |
| `IO` | Waiting on a read/write |
| `Client` | Waiting on the client (common in `psql`) |
| `IPC` | Waiting on another process |

```toml
# postgresql.conf
log_lock_waits = on       # log waits longer than deadlock_timeout
deadlock_timeout = 1s     # delay before deadlock detection
```

## Code examples

### Set a lock timeout for DDL

```sql
-- Fail instead of blocking behind a long transaction
SET lock_timeout = '2s';
ALTER TABLE orders ADD COLUMN note text;
```

`lock_timeout` bounds time spent waiting for a lock; `statement_timeout` bounds total statement time. Use both. Retry a failed DDL after a short delay rather than holding an exclusive lock queue that blocks later readers.

### Do not run heavy DDL at peak

```sql
-- Blocks writes; take ACCESS EXCLUSIVE, which conflicts with everything
ALTER TABLE orders ALTER COLUMN status TYPE text;
```

Prefer online alternatives:

```sql
CREATE INDEX CONCURRENTLY idx_orders_status ON orders (status);
REINDEX INDEX CONCURRENTLY idx_orders_status;
```

`CREATE INDEX CONCURRENTLY` takes `ShareUpdateExclusive`, which does not block ordinary reads/writes, but takes longer and cannot run inside a transaction block.

### Queue processing with `SKIP LOCKED`

```sql
-- SLOW: every worker waits on the same locked row
SELECT * FROM jobs WHERE status = 'pending' ORDER BY id LIMIT 1 FOR UPDATE;

-- FAST: each worker takes a different row
SELECT * FROM jobs
WHERE status = 'pending'
ORDER BY id
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

`SKIP LOCKED` is appropriate for queue-like tables. Do not use it when every consumer must see every row.

### Consistent lock ordering prevents deadlocks

```sql
-- Deadlock-prone: T1 locks 1 then 2, T2 locks 2 then 1
BEGIN;
UPDATE accounts SET amount = amount - 100 WHERE id = 1;
UPDATE accounts SET amount = amount + 100 WHERE id = 2;
COMMIT;
```

Lock rows in a deterministic order (for example ascending `id`) in every transaction, or use a single statement that updates both rows. Retry transactions that fail with a deadlock (`SQLSTATE 40P01`).

### Idle-in-transaction sessions

```sql
-- Find sessions holding snapshots and locks while doing nothing
SELECT pid, state, now() - state_change AS idle_time, xact_start, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY state_change;
```

```toml
# postgresql.conf
idle_in_transaction_session_timeout = '5min'
```

An idle transaction holds its snapshot, blocks vacuum, and can hold locks. Keep transactions short and set a timeout.

### Advisory locks

```sql
-- Only one worker runs a migration at a time
SELECT pg_try_advisory_lock(12345);

SELECT pg_advisory_unlock(12345);
```

Advisory locks are application-defined; they do not protect tables on their own. Prefer transaction-scoped `pg_advisory_xact_lock` when the lock should be released automatically at commit.

### Detect and diagnose deadlocks

```sql
SELECT datname, deadlocks FROM pg_stat_database;
```

Read the server log entry: it lists the processes, the locks each holds, and the statement that triggered the check. The `deadlock_timeout` delay avoids paying for detection on every wait.

### Serializable isolation

Serializable transactions use **predicate locks** and can abort with `40001` (serialization failure). That is expected, not a bug: retry the transaction. Keep serializable transactions short and free of external calls.

## Anti-patterns

1. Long transactions or idle-in-transaction sessions holding locks and snapshots.
2. DDL at peak without `lock_timeout` or an online alternative.
3. `SELECT FOR UPDATE` on a queue without `SKIP LOCKED`.
4. Inconsistent lock ordering across transactions.
5. Catching a deadlock and not retrying the transaction.
6. Using `LOCK TABLE` unnecessarily or at a stronger mode than needed.
7. Assuming MVCC means writers never block; row conflicts and DDL still block.
8. Ignoring `log_lock_waits` and `pg_stat_database.deadlocks`.
9. Holding an advisory lock without a clear release path.
10. Setting `statement_timeout` but not `lock_timeout` for DDL.

## Checklist

- Is a session blocked, or merely slow? Check `wait_event_type`.
- Which pid blocks it (`pg_blocking_pids`), and what is that session doing?
- Is `lock_timeout` set for DDL and migrations?
- Are transactions short, with no idle-in-transaction sessions?
- Is lock ordering consistent across code paths?
- Are queue consumers using `SKIP LOCKED` where appropriate?
- Are deadlocks retried at the application layer?
- Is `log_lock_waits` enabled for diagnosis?
- Is serializable isolation retried on `40001`?
- Are online DDL variants used for large tables?

## Related

- `write-latency.md` for commit and checkpoint stalls.
- `tables.md` for vacuum blocked by long transactions.
- `scans-joins.md` for plans that increase lock duration.
- `planner.md` for `SELECT FOR UPDATE` plans and row locking.
