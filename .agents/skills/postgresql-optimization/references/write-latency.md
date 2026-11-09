# Write Latency: WAL, Commits, and Checkpoints

## Description

Why writes are slow or spiky. Every change to a page is first recorded in the **write-ahead log (WAL)**, then applied to the buffer cache and written to disk later. A commit is durable only after the WAL up to its commit record is flushed. Write latency is therefore dominated by WAL fsync, commit batching, checkpoint behavior, and write amplification from indexes. WAL is written sequentially, so separating it on fast storage helps.

## When to use

- `COMMIT` latency is high, or throughput collapses under concurrent writes.
- Write latency spikes periodically (checkpoints) instead of staying flat.
- WAL grows unexpectedly or fills the disk.
- Choosing `synchronous_commit`, `commit_delay`, or checkpoint settings.
- Adding indexes to a write-heavy table.
- Replication slots or logical decoding retain WAL.

## Checks

```sql
SHOW synchronous_commit;
SHOW commit_delay;
SHOW commit_siblings;
SHOW wal_buffers;
SHOW checkpoint_timeout;
SHOW max_wal_size;
SHOW checkpoint_completion_target;
SHOW full_page_writes;
SHOW wal_level;

-- PostgreSQL 14: WAL and checkpoint counters
SELECT * FROM pg_stat_wal;
SELECT checkpoints_timed, checkpoints_req,
       buffers_checkpoint, buffers_clean, maxwritten_clean,
       buffers_backend, buffers_backend_fsync
FROM pg_stat_bgwriter;

-- Current WAL position and size
SELECT pg_current_wal_lsn(), pg_current_wal_insert_lsn();
SELECT pg_size_pretty(sum(size)) AS wal_size
FROM pg_ls_waldir();
```

| Symptom | Likely cause | First check |
|---|---|---|
| High commit latency, low CPU | Synchronous WAL fsync per commit | `synchronous_commit`, storage latency |
| Periodic latency spikes | Checkpoint flushing dirty pages | `checkpoints_req`, `max_wal_size` |
| WAL grows without bound | Replication slot / archiver stuck | `pg_replication_slots`, archive status |
| Writes slower after adding indexes | Write amplification | Index count and `idx_tup_write` |
| `buffers_backend_fsync` high | Backends forced to fsync their own writes | Checkpoint tuning, I/O capacity |

## Code examples

### Understand the commit path

```sql
BEGIN;
UPDATE accounts SET amount = amount - 100 WHERE id = 1;
COMMIT;   -- waits for WAL flush when synchronous_commit = on
```

The page change is in the buffer cache; the WAL record is in WAL buffers, then written and fsynced. `synchronous_commit = on` (default) makes `COMMIT` wait for that flush. Disabling it returns immediately but can lose recent committed transactions on a crash.

### Relax durability only where acceptable

```sql
-- Global (postgresql.conf): risky for financial data
-- synchronous_commit = off

-- Per transaction: only this transaction may be lost on crash
BEGIN;
SET LOCAL synchronous_commit = off;
INSERT INTO audit_log (...) VALUES (...);
COMMIT;
```

Classify transactions: critical writes keep synchronous commit; analytics, logs, or re-derivable data may use asynchronous commit. Document the loss window: with `wal_writer_delay = 200ms`, roughly the last 0.6 seconds of commits can be lost.

### Batch commits

```toml
# postgresql.conf
commit_delay = 0          # microseconds; 0 = no delay
commit_siblings = 5       # only delay when this many transactions are active
```

`commit_delay` makes a committing backend pause briefly when at least `commit_siblings` transactions are active, so several commits can share one fsync. It helps only commit-heavy workloads with many short transactions; it adds latency to each commit by design.

### Tune checkpoints

```toml
# postgresql.conf
max_wal_size = 4GB                 # more WAL between checkpoints
checkpoint_timeout = 15min         # less frequent checkpoints
checkpoint_completion_target = 0.9 # spread writes over most of the interval
```

Frequent checkpoints (`checkpoints_req` high) mean `max_wal_size` is too small for the write rate. Checkpoints spread dirty-page writes over time so the system does not stall at the end. Monitor `buffers_checkpoint` and `buffers_backend` to see whether backends are being forced to write.

### Manual checkpoint for measurement

```sql
CHECKPOINT;
SELECT * FROM pg_stat_bgwriter;
```

Use only for diagnostics or controlled maintenance; frequent manual checkpoints add I/O.

### Watch replication slots

```sql
SELECT slot_name, plugin, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots;
```

An inactive slot retains WAL forever. A lagging replica also retains WAL. Both can fill the disk and increase checkpoint pressure. Drop unused slots.

### Reduce write amplification

```sql
-- Every index adds an entry for each new row version
SELECT relname,
       pg_size_pretty(pg_indexes_size(relid)) AS index_bytes,
       (SELECT count(*) FROM pg_index i WHERE i.indrelid = relid) AS index_count
FROM pg_stat_user_tables
ORDER BY pg_indexes_size(relid) DESC
LIMIT 20;
```

For update-heavy tables, fewer indexes mean less WAL and fewer page writes. Remove unused and duplicate indexes; consider partial indexes for the hot subset.

### WAL on separate storage

WAL writes are sequential; placing `pg_wal` on a fast, separate device (or a symlink to a mounted volume) can isolate it from random data-file I/O. This is an operational choice; measure before and after.

## Anti-patterns

1. `fsync = off` in production; it risks corruption, not just lost transactions.
2. `synchronous_commit = off` for data that must survive a crash.
3. Very small `max_wal_size`, causing constant checkpoints.
4. `checkpoint_completion_target` near 0, concentrating checkpoint I/O.
5. Many redundant indexes on an update-heavy table.
6. Long transactions that hold WAL and old snapshots.
7. Ignoring inactive replication slots and `pg_wal` growth.
8. Disabling `full_page_writes` without understanding torn-page risk.
9. Adding indexes to speed reads without measuring write latency.
10. Concluding "disk is slow" before checking `checkpoints_req` and `buffers_backend_fsync`.

## Checklist

- Is `synchronous_commit` appropriate for each class of transaction?
- Is commit latency dominated by fsync, locks, or checkpoint I/O?
- Are checkpoints timed or requested? What is `max_wal_size` relative to the write rate?
- Is `checkpoint_completion_target` spreading writes?
- Are backends forced to fsync (`buffers_backend_fsync`)?
- Are replication slots active and is WAL retained under control?
- Is the index count justified for the write rate?
- Is `full_page_writes` left on unless there is a proven, documented reason?
- Is WAL on storage that matches its sequential write pattern?
- Was the change validated with a write benchmark (for example `pgbench`) under concurrency?

## Related

- `cache.md` for dirty buffers and eviction.
- `tables.md` for vacuum, bloat, and write amplification.
- `locks.md` for lock waits that masquerade as write latency.
- `index-types.md` for write costs per index type.
