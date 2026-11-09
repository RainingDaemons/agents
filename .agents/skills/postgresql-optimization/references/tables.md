# Table and Index Bloat

## Description

Why a table or index is larger than its live data, and how to reclaim the space safely. MVCC keeps old row versions until vacuum removes them; `UPDATE` and `DELETE` leave dead tuples that occupy pages and inflate sequential-scan cost. Indexes accumulate entries for dead versions and grow with writes. The fixes are vacuum/autovacuum health, `fillfactor` and HOT updates, and online rebuilds.

## When to use

- `pg_relation_size` is far larger than `n_live_tup` suggests.
- Sequential scans are slow even though the live row count is modest.
- Index-only scans report `Heap Fetches > 0` or keep fetching from the heap.
- Autovacuum runs constantly, falls behind, or never runs on a large table.
- Indexes grow without a matching growth in rows.
- After bulk deletes or a large migration.

## Checks

```sql
-- Dead tuples and maintenance history
SELECT relname, n_live_tup, n_dead_tup,
       last_vacuum, last_autovacuum,
       last_analyze, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Table and index sizes
SELECT schemaname, relname,
       pg_size_pretty(pg_relation_size(relid))            AS heap,
       pg_size_pretty(pg_indexes_size(relid))             AS indexes,
       pg_size_pretty(pg_total_relation_size(relid))      AS total
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 20;

-- Exact bloat (requires pgstattuple)
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT * FROM pgstattuple('orders');
SELECT * FROM pgstatindex('orders_user_date_idx');

-- Vacuum progress (PostgreSQL 12+)
SELECT * FROM pg_stat_progress_vacuum;
```

Bloat signals:

| Signal | Meaning |
|---|---|
| `n_dead_tup` high and growing | Vacuum cannot keep up or is blocked |
| `relpages` high vs `n_live_tup` | Dead versions and free space inflate the file |
| `Heap Fetches > 0` | Visibility map not set; vacuum is behind |
| `pgstattuple.dead_tuple_percent` high | Dead tuples occupy real space |
| `pgstatindex` leaf density low | Index bloat (empty/dead entries) |
| Old `xmin` horizon | A long transaction or replication slot blocks cleanup |

## Code examples

### Read the health of an update-heavy table

```sql
SELECT relname,
       n_live_tup,
       n_dead_tup,
       round(100.0 * n_dead_tup / nullif(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
       last_autovacuum,
       last_autoanalyze
FROM pg_stat_user_tables
WHERE relname = 'orders';
```

If `dead_pct` is high and `last_autovacuum` is old, find out why before tuning: a long transaction, an idle-in-transaction session, a prepared transaction, or a replication slot can hold back the oldest xmin.

```sql
-- Sessions that can hold back vacuum
SELECT pid, state, xact_start, now() - xact_start AS age, query
FROM pg_stat_activity
WHERE state IN ('active', 'idle in transaction')
  AND xact_start IS NOT NULL
ORDER BY xact_start
LIMIT 10;
```

### Tune autovacuum per large table

The defaults `50 + 0.2 * reltuples` (vacuum) and `50 + 0.1 * reltuples` (analyze) are too high for very large tables: 20% dead tuples means a lot of bloat before cleanup starts.

```sql
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.02,
    autovacuum_analyze_scale_factor = 0.01
);
```

Tune per table; raising the global scale factor affects every table.

### Manual maintenance

```sql
VACUUM (ANALYZE) orders;              -- reclaim dead tuples + refresh stats
ANALYZE orders;                       -- statistics only
VACUUM FULL orders;                   -- rewrite, ACCESS EXCLUSIVE lock
REINDEX INDEX CONCURRENTLY orders_user_date_idx;  -- online index rebuild
```

`VACUUM FULL` rewrites the table under an exclusive lock and is usually the wrong choice for a busy system. Prefer `pg_repack` for online table reclaim and `REINDEX CONCURRENTLY` for index bloat.

### Enable HOT updates with `fillfactor`

```sql
ALTER TABLE orders SET (fillfactor = 85);
```

A lower fillfactor leaves room on each page for updated versions. A HOT update stays on the same page and does not add index entries, but only when:

- The new version fits on the same page.
- No indexed column changed.
- The page has enough free space.

Do not index volatile columns unnecessarily; every indexed column change defeats HOT.

### Confirm table bloat before rebuilding

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;

SELECT table_len,
       tuple_count,
       dead_tuple_count,
       round(dead_tuple_percent, 2)  AS dead_pct,
       free_percent
FROM pgstattuple('orders');
```

Rebuild only when the dead/free space justifies the lock and I/O. A small amount of free space is normal and useful for future updates.

### Index bloat

```sql
SELECT * FROM pgstatindex('orders_user_date_idx');
SELECT pg_size_pretty(pg_relation_size('orders_user_date_idx'));
```

Index bloat accumulates from updates and deletes. `REINDEX INDEX CONCURRENTLY` rebuilds without blocking writes (it cannot run inside a transaction block and takes longer). Verify that the index is still needed before rebuilding: an unused index is better dropped.

```sql
SELECT s.indexrelname, s.idx_scan,
       pg_size_pretty(pg_relation_size(s.indexrelid)) AS size
FROM pg_stat_user_indexes s
JOIN pg_index i ON i.indexrelid = s.indexrelid
WHERE s.idx_scan = 0
  AND NOT i.indisunique
  AND NOT i.indisprimary
ORDER BY pg_relation_size(s.indexrelid) DESC;
```

### Freezing and wraparound

```sql
SELECT relname, age(relfrozenxid) AS xid_age
FROM pg_class
WHERE relkind = 'r'
ORDER BY xid_age DESC
LIMIT 10;
```

Monitor `xid_age` against `autovacuum_freeze_max_age` (default 200 million). A table approaching 2 billion needs freezing; wraparound protection forces aggressive vacuum.

## Anti-patterns

1. `VACUUM FULL` on a live system without planning for the exclusive lock and disk space.
2. Disabling autovacuum on a busy table.
3. Tuning the global autovacuum scale factor instead of per-table thresholds.
4. Keeping long transactions or idle-in-transaction sessions open; they block cleanup.
5. Rebuilding an index that no query uses.
6. Indexing volatile columns that defeat HOT updates.
7. Setting `fillfactor = 100` on an update-heavy table.
8. Assuming `n_dead_tup` is exact; it is an estimate between analyzes.
9. Ignoring replication slots that retain WAL and old snapshots.
10. Running `REINDEX` (non-concurrent) during peak hours.

## Checklist

- Is `n_dead_tup` growing faster than autovacuum can clean?
- Is some session or replication slot holding back the oldest xmin?
- Is the table update-heavy? Is `fillfactor` tuned and HOT possible?
- Are indexes justified by query usage and not bloated?
- Are `VACUUM`/`ANALYZE` running after bulk loads?
- Is `xid_age` under control for wraparound?
- Is the rebuild method online (`pg_repack`, `REINDEX CONCURRENTLY`) where needed?
- Is there enough disk space for a rewrite?
- Did you verify with `pgstattuple`/`pgstatindex` before rebuilding?

## Related

- `data.md` for pages, tuples, and MVCC versions.
- `cache.md` for how bloat affects buffer usage.
- `write-latency.md` for WAL and checkpoint pressure.
- `index-types.md` for index-specific write costs.
