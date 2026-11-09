---
name: postgresql-optimization
description: SQL performance work in PostgreSQL - EXPLAIN/ANALYZE, index strategy (composite, partial, expression, covering, GIN/GiST/BRIN), planner and join analysis, statistics, MVCC/vacuum bloat, pagination, aggregation rewrites, locks, WAL, and connection pooling. Use when a query is slow or an index strategy is needed.
license: MIT
metadata:
  author: rainingdaemons/agents
  version: "1.0"
  source: Generated using PostgreSQL Optimization recommendations.
---

# PostgreSQL Optimization

How to make PostgreSQL queries fast. This skill owns **query performance, indexing, and the operational causes behind slow plans**. The engine explains *why* a plan is slow: pages, tuples, dead rows, statistics, access paths, join and sort algorithms, caches, WAL, and locks.

Engine-specific types/functions live in `postgresql-patterns`; schema/normalization in `database-design`; repo rules in `internal/conventions/DATABASE_CONVENTIONS.md`.

## When to activate

- A query is slow, a plan looks wrong, or estimates do not match reality.
- Choosing, reviewing, or removing indexes.
- Rewriting a query (joins, grouping, pagination, CTEs, subqueries).
- Tables or indexes are bloated, autovacuum is behind, or index-only scans fetch from the heap.
- Maintaining statistics and reclaiming space.
- Diagnosing lock waits, write latency, checkpoint pressure, or parallel-plan limits.

## Where to start

The guides in `references/` are the source material for this skill. Each is a short, task-oriented note with diagnostic queries, `EXPLAIN` examples, anti-patterns, and a checklist; verify version-specific behavior before applying it to another major version.

| Question | Read |
|---|---|
| How are rows stored, updated, and deleted? | [data.md](references/data.md) |
| Why is the table/index bloated? | [tables.md](references/tables.md) |
| Why does the planner choose this plan? | [planner.md](references/planner.md) |
| Why is this scan or join slow? | [scans-joins.md](references/scans-joins.md) |
| Which index type should I use? | [index-types.md](references/index-types.md) |
| Why is the cache not helping? | [cache.md](references/cache.md) |
| Why is write latency high? | [write-latency.md](references/write-latency.md) |
| Why is the query blocked? | [locks.md](references/locks.md) |
| Why did parallel execution not help? | [parallel.md](references/parallel.md) |

## Workflow

1. **Reproduce and measure.** Capture the slow query, parameters, table sizes, row counts, and relevant settings. Establish a baseline with `EXPLAIN (ANALYZE, BUFFERS)` and `pg_stat_statements`.
2. **Read the plan from the inside out.** Find the node with the largest actual time or the largest row-estimate error. Identify the access path, join method, sort/hash spills, and buffer/temp I/O.
3. **Explain the cause with internals.** Decide whether the problem is statistics, storage/bloat, a missing or unsuitable access path, a join/sort algorithm, a lock wait, a cache miss, or configuration.
4. **Change one thing.** Prefer statistics and maintenance before new indexes; prefer query rewrites before configuration; prefer algorithmic fixes before hardware.
5. **Verify.** Re-run `EXPLAIN (ANALYZE, BUFFERS)` on the same data and settings. Compare estimates, actual rows, buffers, temp blocks, and timing. Do not accept a win that is not measured.

## Read the plan first

Always measure before changing anything.

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS)
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.created_at >= '2024-01-01'::date
  AND u.created_at <  '2025-01-01'::date
GROUP BY u.id, u.name;
```

`EXPLAIN ANALYZE` executes the statement. For `UPDATE`, `DELETE`, or `INSERT`, wrap it in `BEGIN; ... ROLLBACK;` unless you intend the change to persist. `ANALYZE` adds instrumentation overhead; compare plans captured under the same conditions, and warm caches first when measuring steady-state performance.

### Plan fields

| Field | Meaning | What to check |
|---|---|---|
| `cost=startup..total` | Planner's abstract cost | Relative only; not milliseconds |
| `rows` | Estimated rows for the node | Compare with `actual rows` |
| `width` | Estimated average row width | Large widths increase sort/hash memory |
| `actual time` | Measured startup and total time | Look for the node with the largest share |
| `loops` | Times the node ran | High loops on an inner node can dominate |
| `actual rows` | Rows returned per loop | Multiply by `loops` to compare globally |
| `Buffers: shared hit/read` | Pages found in / read into shared buffers | High `read` means cache misses or a large scan |
| `Buffers: temp` | Temporary blocks for sorts/hashes | Any temp use means a spill |
| `Rows Removed by Filter` | Rows read but discarded | High values mean a non-selective scan |
| `Heap Fetches` | Heap visits during an index-only scan | Non-zero means the visibility map is incomplete |
| `Sort Method` | `quicksort`, `top-N heapsort`, `external merge` | `external merge` means `work_mem` was exceeded |
| `Hash Batches` | Number of hash partitions | More than 1 means the hash table spilled |
| `Memory Usage` | Memory used by hash/sort | Compare with `work_mem` |
| `Recheck Cond` | Rows rechecked after a lossy index scan | High values often mean BRIN/GiST lossiness |
| `Workers Planned/Launched` | Parallel workers | Launched below planned means limits or contention |

### Plan nodes

| Node | What it does | Typical use |
|---|---|---|
| `Seq Scan` | Reads the whole table | Low selectivity or whole-table work |
| `Index Scan` | Index lookup, then heap fetch | High selectivity |
| `Index Only Scan` | Returns data from the index | Covering index + visibility map |
| `Bitmap Index Scan` / `Bitmap Heap Scan` | Collects TIDs, reads heap pages in order | Medium selectivity or poor correlation |
| `Nested Loop` | For each outer row, probes the inner side | Small outer set with an indexed inner side |
| `Hash Join` / `Hash` | Builds a hash table on the inner side | Large equality joins |
| `Merge Join` | Merges sorted inputs | Large joins already sorted, or inequality joins |
| `Sort` | Sorts rows | `ORDER BY`, merge joins, grouping |
| `Aggregate` / `HashAggregate` / `GroupAggregate` | Aggregates rows | `GROUP BY`, `DISTINCT` |
| `Gather` / `Gather Merge` | Collects parallel results | Parallel plans |
| `Materialize` | Caches inner rows for reuse | Repeated inner-side access |
| `Memoize` | Caches parameterized inner results | Repeated nested-loop lookups (v14+) |
| `CTE Scan` | Reads a CTE result | Materialized CTEs |

### Common red flags

| Symptom | Likely cause | First checks |
|---|---|---|
| Seq Scan on a large table returning few rows | Missing or ineffective index; low selectivity; stale stats; bloat | Predicate sargability, index, `ANALYZE`, `VACUUM` |
| Index Scan with many buffers or rows | Low correlation; non-selective index; bloat; too many heap fetches | Bitmap scan, covering index, `VACUUM`, `CLUSTER` |
| Index Only Scan with `Heap Fetches > 0` | Visibility map not set | `VACUUM` and autovacuum health |
| Nested Loop with high `loops` | Underestimated outer cardinality; missing inner index | Statistics, extended stats, inner index, rewrite |
| `Hash Batches > 1` | Hash table exceeded `work_mem` | Increase `work_mem`; select fewer/narrower columns |
| `Sort Method: external merge` | Sort exceeded `work_mem` | Increase `work_mem`; use an index order; `LIMIT` |
| Estimated vs actual rows differ by 10x or more | Stale or insufficient statistics; correlated predicates | `ANALYZE`, `default_statistics_target`, extended statistics |
| Table/index size far above live rows | Dead tuples or bloat | Autovacuum tuning, `VACUUM FULL`/`pg_repack`, `REINDEX` |
| High `shared read`, low `shared hit` | Working set does not fit cache | `shared_buffers`, `effective_cache_size`, query shape |
| `Workers Launched < Workers Planned` | Worker limits or contention | `max_parallel_workers`, `max_worker_processes` |
| Lock waits or deadlocks | Long transactions, DDL, row lock conflicts | `pg_locks`, `pg_stat_activity`, `lock_timeout` |
| High write latency | Synchronous commit, checkpoints, too many indexes | WAL settings, `max_wal_size`, index count |
| `temp_blks_read/written > 0` | Sort/hash spill | `work_mem`; reduce row width; index order |

### System-wide view

```sql
SELECT query,
       calls,
       total_exec_time,
       mean_exec_time,
       rows,
       shared_blks_hit,
       shared_blks_read,
       temp_blks_read,
       temp_blks_written
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

In PostgreSQL 12 and earlier, use `total_time` and `mean_time`. On PostgreSQL 13+, `total_exec_time` and `mean_exec_time` are the execution-time columns.

## Storage and MVCC

Understanding the physical layout explains most "the index exists but the query is still slow" cases.

- Tables and indexes are stored in 8 KB pages. A heap page holds tuples plus item pointers; an index entry points to a tuple's TID (`page, offset`).
- `UPDATE` writes a new tuple version and marks the old one dead; it does not overwrite in place. `DELETE` marks tuples dead. Dead versions remain until vacuumed.
- Dead tuples leave reusable gaps. If they are not reclaimed, new inserts can extend the table while old pages remain sparsely filled, so bloat increases the number of pages. Sequential-scan cost uses `relpages`, not the live row count, so bloat directly inflates scan cost.
- Indexes accumulate entries for old tuple versions too. More indexes mean more write amplification on every `UPDATE`.
- `fillfactor` reserves free space per page for updates. A lower fillfactor can enable **HOT updates** (same page, no new index entries) when the updated columns are not indexed.
- **Index-only scans** need the visibility map: a page is all-visible only after `VACUUM` marks it. Otherwise the executor visits the heap (`Heap Fetches`).
- **TOAST** stores large values out of line. Selecting large `text`, `jsonb`, or `bytea` columns increases I/O and memory even if the application does not need them.
- Long transactions, idle-in-transaction sessions, and logical replication slots can hold back the oldest xmin, preventing vacuum from cleaning dead tuples.

Practical consequences:

```sql
-- Update-heavy table: leave room for HOT updates
ALTER TABLE orders SET (fillfactor = 85);

-- Partial index for the hot subset, not the whole table
CREATE INDEX idx_orders_pending ON orders (created_at)
WHERE status = 'pending';

-- Covering index: include columns needed by the query but not used for search
CREATE INDEX idx_orders_covering ON orders (user_id, status)
INCLUDE (total, created_at);
```

- Use `fillfactor` below 100 only for tables with frequent updates; it trades space for fewer index updates and less bloat.
- Do not index volatile columns unless queries need them; every indexed column change defeats HOT.
- For append-only tables, focus on page fill, index growth, and tuple freezing rather than dead tuples.

## Vacuum, autovacuum, and analyze

`VACUUM` reclaims dead tuple space and updates the visibility map. `ANALYZE` refreshes planner statistics. Autovacuum does both based on table activity.

Autovacuum triggers (PostgreSQL 14 defaults):

- Vacuum when `n_dead_tup > autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor * reltuples` (`50 + 0.2 * reltuples`).
- Vacuum on inserts when `n_ins_since_vacuum > autovacuum_vacuum_insert_threshold + autovacuum_vacuum_insert_scale_factor * reltuples` (`1000 + 0.2 * reltuples`, v13+).
- Analyze when `n_mod_since_analyze > autovacuum_analyze_threshold + autovacuum_analyze_scale_factor * reltuples` (`50 + 0.1 * reltuples`).

For large tables the default 20% dead-tuple scale factor is often too high. Tune per table instead of globally:

```sql
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.02,
  autovacuum_analyze_scale_factor = 0.01
);
```

Useful maintenance commands:

```sql
VACUUM (ANALYZE) orders;          -- reclaim space + refresh statistics
ANALYZE orders;                   -- statistics only
VACUUM FULL orders;               -- rewrites the table; ACCESS EXCLUSIVE lock
REINDEX INDEX CONCURRENTLY idx_orders_user_date;  -- online index rebuild
```

Monitor health:

```sql
SELECT relname, n_live_tup, n_dead_tup,
       last_vacuum, last_autovacuum,
       last_analyze, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

SELECT * FROM pg_stat_progress_vacuum;

SELECT relname, age(relfrozenxid) AS xid_age
FROM pg_class
WHERE relkind = 'r'
ORDER BY xid_age DESC
LIMIT 10;
```

- `pg_stat_progress_vacuum` exists from PostgreSQL 12.
- Compare `xid_age` with `autovacuum_freeze_max_age` (default 200 million) and watch for wraparound risk near 2 billion.
- `VACUUM FULL` and non-concurrent `REINDEX` take heavy locks and rewrite tables or indexes; use `pg_repack` or `REINDEX CONCURRENTLY` when availability matters.
- Keep transactions short. A long-running transaction or an idle-in-transaction session blocks cleanup and can cause severe bloat.
- Bulk loads should be followed by `ANALYZE`; use `VACUUM` when the load creates dead tuples or when index-only scans matter.

## Statistics and cardinality

The planner chooses access paths and join orders from estimates. Bad estimates are the most common cause of bad plans.

Inspect statistics:

```sql
SELECT attname, n_distinct, null_frac, avg_width, correlation
FROM pg_stats
WHERE schemaname = 'public' AND tablename = 'orders';
```

Key fields:

- `n_distinct`: estimated distinct values; negative values mean a fraction of the row count.
- `null_frac`: fraction of NULLs.
- `most_common_vals` / `most_common_freqs`: MCV list for skewed values.
- `histogram_bounds`: value distribution outside the MCV list.
- `correlation`: physical vs logical order; close to 1 or -1 favors index scans, near 0 favors bitmap or sequential scans.
- `avg_width`: row width used for sort/hash memory estimates.

Improve estimates:

```sql
-- More detail for a skewed column
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
ANALYZE orders;

-- Multivariate statistics for correlated predicates
CREATE STATISTICS orders_user_status (dependencies)
ON user_id, status FROM orders;

CREATE STATISTICS orders_nd (ndistinct)
ON user_id, status FROM orders;

CREATE STATISTICS orders_mcv (mcv)
ON user_id, status FROM orders;

ANALYZE orders;
```

- The planner assumes predicates are independent. Correlated columns underestimate cardinality; extended statistics fix this.
- Functions on columns defeat column statistics. Use an expression index and expression statistics, or rewrite the predicate to be sargable.
- Expression statistics can be collected with `CREATE STATISTICS ... ON (expression) FROM table;` (PostgreSQL 14+); the query must use the same expression form.
- `default_statistics_target` (default 100) controls detail; raise it for skewed/high-cardinality columns, not for every column.
- `pg_statistic_ext` and `pg_stats_ext` expose extended statistics.
- Do not "fix" bad plans with `enable_*` settings permanently. Fix statistics, indexes, or the query first.

## Access paths

### Sequential scan

Best when the query needs most of the table or the predicate is not selective. Cost is roughly `relpages * seq_page_cost + reltuples * cpu_tuple_cost`; bloat increases the first term. Parallel sequential scans are used for large tables.

### Index scan

Best for selective predicates. It reads index pages, then fetches heap tuples, which can be random I/O. Cost depends on `random_page_cost` and on `correlation`. On SSDs, lower `random_page_cost` (for example, 1.1–2.0) so the planner values index access more realistically; keep `seq_page_cost` as the reference.

### Index-only scan

Requires an index that covers all referenced columns and a visibility map that marks heap pages all-visible. Check `Heap Fetches`: zero is the goal. Vacuum regularly; autovacuum must keep up on insert-only tables too.

### Bitmap scan

Collects TIDs, sorts them by heap page, then reads pages in order. Good for medium selectivity and low correlation. A lossy bitmap rechecks pages or tuples; watch `Recheck Cond` and `Rows Removed by Index Recheck`.

### Parallel plans

- `min_parallel_table_scan_size` (default 8 MB) gates parallel scans.
- `max_parallel_workers_per_gather` (default 2) limits workers per plan.
- `max_parallel_workers` (default 8) and `max_worker_processes` (default 8) are global limits.
- Workers may be planned but not launched under contention. Parallelism helps large scans and aggregations, not small OLTP queries or write-heavy work.

## Join methods

| Method | Best when | Watch for |
|---|---|---|
| Nested Loop | Outer side is small and the inner side has an efficient index | High `loops`, repeated inner scans, missing inner index, underestimated outer cardinality |
| Hash Join | Equality join with a large inner side | `Hash Batches > 1`, `Memory Usage` near `work_mem`, wide rows |
| Merge Join | Inputs already sorted, large equi-joins, or inequality joins | Extra `Sort` nodes, `external merge`, stale stats producing a bad sort order |

- Hash joins usually build the hash table on the smaller side and use `work_mem * hash_mem_multiplier` (default 1.0). Reducing selected columns reduces hash memory.
- `SELECT *` increases row width and can turn a one-batch hash join into a multi-batch spill.
- Nested loops rely on the inner side being indexed and on accurate outer cardinality. If estimates are wrong, the planner may choose a nested loop that executes far too many iterations.
- Join order follows estimates. Fix statistics before forcing join order or disabling join methods.

## Sorting and grouping

- `work_mem` is per sort/hash operation, not per connection. A complex query can use it multiple times; concurrency multiplies memory use.
- In-memory sorts use quicksort; `LIMIT` can use top-N heapsort; larger sorts use external merge and spill to disk (`Sort Method: external merge`).
- An index in the right order can eliminate a sort, especially for `ORDER BY ... LIMIT`.
- `HashAggregate` uses `work_mem`; `GroupAggregate` needs sorted input and can benefit from an index.
- Large `DISTINCT` or `GROUP BY` operations are memory-hungry. Pre-aggregate, reduce columns, or use a partial/expression index where appropriate.
- Window functions often require a sort; an index matching `PARTITION BY`/`ORDER BY` can help.

## Index strategies

```sql
-- Composite index: equality columns first, then range/sort columns
CREATE INDEX idx_orders_user_date ON orders (user_id, order_date);

-- Partial index: only the rows the query touches
CREATE INDEX idx_active_users ON users (created_at) WHERE status = 'active';

-- Expression index: for computed predicates
CREATE INDEX idx_users_lower_email ON users (lower(email));

-- Covering index: avoid heap lookups with INCLUDE
CREATE INDEX idx_orders_covering ON orders (user_id, status) INCLUDE (total, created_at);

-- Pattern matching under a non-C collation
CREATE INDEX idx_users_email_pattern ON users (email text_pattern_ops);

-- Trigram index for leading wildcards and ILIKE (requires pg_trgm)
CREATE INDEX idx_users_name_trgm ON users USING gin (name gin_trgm_ops);

-- JSONB containment
CREATE INDEX idx_events_payload ON events USING gin (payload jsonb_path_ops);

-- BRIN for large append-only, naturally ordered data
CREATE INDEX idx_metrics_time_brin ON metrics USING brin (recorded_at)
WITH (pages_per_range = 32);
```

### Index type selection

| Index | Best for | Supports | Limits |
|---|---|---|---|
| B-tree | Equality, ranges, sorting, uniqueness | Multicolumn, `INCLUDE`, index-only, backward scan, `NULLS FIRST/LAST`, expression/partial indexes | Not for containment, JSONB, arrays, full-text, or nearest-neighbor queries |
| Hash | Equality only | Equality, bitmap scans | No ordering, uniqueness, multicolumn, `INCLUDE`, or index-only scans; rarely better than B-tree |
| GiST | Geometry, ranges, full-text, nearest neighbor with an operator class | Distance ordering with KNN operators; lossy recheck | Operator-class dependent; rechecks; larger and slower than B-tree for simple equality |
| SP-GiST | Points, text, space-partitioned data | Partitioned search structures | Niche; operator-class dependent |
| GIN | JSONB, arrays, `tsvector`, trigrams | Containment, existence, full-text search | Expensive writes; pending list; no index-only scans |
| BRIN | Very large append-only tables with correlated data | Min/max summaries; tiny index | Lossy; poor for random point lookups; requires physical correlation |

- `jsonb_path_ops` is smaller and faster but supports fewer operators than the default `jsonb_ops`; choose by query operators.
- GIN `fastupdate` and its pending list trade write cost for read cost; check `gin_pending_list_limit` and clean the pending list when needed.
- BRIN's `pages_per_range` controls the summary granularity; smaller ranges mean more index entries but less rechecking.
- `btree_gist` enables GiST indexes on ordinary scalar types for exclusion constraints and multicolumn mixes.

### Index rules of thumb

- Add indexes for foreign keys and for columns in `WHERE`, `ORDER BY`, and `JOIN` predicates.
- Composite column order matters: equality columns first, then range/sort columns. A B-tree cannot efficiently use a later column without the leading columns.
- Match `ORDER BY` direction and null ordering in the index definition when the query depends on it.
- A partial index predicate must be implied by the query predicate for the planner to use it.
- An expression index is used only when the expression matches exactly. Keep statistics for expression indexes current.
- Covering indexes can make index-only scans possible; `INCLUDE` columns cannot be used for search or ordering.
- More indexes cost write throughput and space. Remove unused and duplicate indexes:

```sql
SELECT s.schemaname, s.relname AS table_name, s.indexrelname AS index_name,
       s.idx_scan, s.idx_tup_read, s.idx_tup_fetch,
       pg_size_pretty(pg_relation_size(s.indexrelid)) AS size
FROM pg_stat_user_indexes s
JOIN pg_index i ON i.indexrelid = s.indexrelid
WHERE s.idx_scan = 0
  AND NOT i.indisunique
  AND NOT i.indisprimary
ORDER BY pg_relation_size(s.indexrelid) DESC;
```

- Index bloat is real: `REINDEX CONCURRENTLY` rebuilds a bloated index; the `pgstattuple`/`pgstatindex` extension can confirm it.
- Create indexes with `CREATE INDEX CONCURRENTLY` in production; it avoids blocking writes but takes longer and cannot run inside a transaction block.

## Query rewrites

```sql
-- BAD: OFFSET pagination degrades on large offsets
SELECT * FROM products ORDER BY id OFFSET 10000 LIMIT 20;

-- GOOD: keyset/cursor pagination
SELECT * FROM products WHERE id > $last_id ORDER BY id LIMIT 20;

-- Prefer EXISTS over COUNT(*) for existence checks
-- BAD
SELECT count(*) > 0 FROM orders WHERE user_id = $1;
-- GOOD
SELECT EXISTS (SELECT 1 FROM orders WHERE user_id = $1);

-- Avoid SELECT *; list the columns you need
-- Batch inserts and use RETURNING instead of a follow-up SELECT
-- Dynamic ORDER BY must use a server-side allowlist, never raw user input
```

- Keep predicates **sargable**: do not wrap indexed columns in functions or casts. Use ranges instead of `date_part`/`date_trunc` on the column; use an expression index if the function is unavoidable.
- Avoid implicit type mismatches; they can prevent index use and distort statistics.
- `LIKE 'prefix%'` can use a B-tree with `text_pattern_ops` (or a C collation); leading wildcards need a trigram GIN/GiST index.
- `OR` across different columns may prevent efficient index use; `UNION ALL` or a bitmap-friendly rewrite can help.
- `NOT IN` with NULLs can return no rows unexpectedly; use `NOT EXISTS`.
- Do not use `DISTINCT` to hide duplicate rows caused by a bad join; fix the join or aggregate correctly.
- `UNION ALL` is cheaper than `UNION` when duplicate elimination is not required.
- Aggregate before joining when it reduces the row count; avoid carrying wide columns through joins.
- Non-recursive CTEs are inlined by default in PostgreSQL 12+; use `MATERIALIZED` or `NOT MATERIALIZED` deliberately.
- Prepared statements can reuse generic plans. Parameter-sensitive queries may get a bad generic plan; inspect or use `plan_cache_mode` (PostgreSQL 12+) when diagnosing.
- Batch writes; avoid N+1 queries; use set-based operations instead of row-by-row loops.

## Maintenance and configuration

### Planner cost model

```sql
SELECT name, setting, unit
FROM pg_settings
WHERE name IN (
  'seq_page_cost', 'random_page_cost', 'cpu_tuple_cost', 'cpu_operator_cost',
  'effective_cache_size', 'work_mem', 'maintenance_work_mem',
  'shared_buffers', 'hash_mem_multiplier'
);
```

- `seq_page_cost` (default 1.0) is the reference unit.
- `random_page_cost` (default 4.0) should be reduced for SSDs; the right value depends on hardware.
- `effective_cache_size` is a planner hint for OS + PostgreSQL cache, not an allocation.
- `hash_mem_multiplier` (default 1.0) multiplies `work_mem` for hash operations.

### Common settings

| Setting | PostgreSQL 14 default | Scope | What it changes |
|---|---|---|---|
| `shared_buffers` | 128 MB | Memory | PostgreSQL buffer cache; start around 25% of RAM and tune |
| `effective_cache_size` | 4 GB | Planner | Assumed OS + PostgreSQL cache; not allocated |
| `work_mem` | 4 MB | Per sort/hash | Spill threshold; concurrency multiplies usage |
| `maintenance_work_mem` | 64 MB | Maintenance | `VACUUM`, `CREATE INDEX`, some DDL |
| `autovacuum_work_mem` | -1 | Autovacuum | Falls back to `maintenance_work_mem` |
| `random_page_cost` | 4.0 | Planner | Random I/O cost; lower for SSD |
| `seq_page_cost` | 1.0 | Planner | Sequential I/O reference |
| `default_statistics_target` | 100 | Statistics | Detail of column statistics |
| `autovacuum_naptime` | 1 min | Autovacuum | Scheduling interval |
| `autovacuum_vacuum_scale_factor` | 0.2 | Autovacuum | Dead-tuple threshold fraction |
| `autovacuum_analyze_scale_factor` | 0.1 | Autovacuum | Analyze threshold fraction |
| `max_parallel_workers_per_gather` | 2 | Parallel | Workers per plan |
| `max_parallel_workers` | 8 | Parallel | Global parallel worker slots |
| `max_worker_processes` | 8 | Parallel | Global background worker slots |
| `synchronous_commit` | on | WAL | Durability vs commit latency |
| `checkpoint_timeout` | 5 min | WAL | Checkpoint frequency |
| `max_wal_size` | 1 GB | WAL | WAL size before a checkpoint is forced |

### WAL and checkpoints

- Synchronous commits wait for WAL fsync; asynchronous commits (`synchronous_commit = off`) reduce latency but can lose recent transactions on failure. Set it per transaction when only some writes need durability.
- `commit_delay` (default 0) can batch commits when at least `commit_siblings` (default 5) transactions are active; it helps only commit-heavy workloads.
- WAL writes are sequential; putting WAL on fast/separate storage helps. Frequent checkpoints and small `max_wal_size` increase I/O; tune `checkpoint_timeout`, `max_wal_size`, and `checkpoint_completion_target`.
- `full_page_writes` protects against torn pages; do not disable it without understanding the corruption risk.
- Replication slots can retain WAL and cause disk growth; monitor `pg_replication_slots`.

### Locks and concurrency

- Long transactions hold an old snapshot, block vacuum, and increase bloat. Keep transactions short; set `idle_in_transaction_session_timeout`.
- DDL takes strong locks (`ACCESS EXCLUSIVE` in many cases) and blocks reads/writes on the table. Set `lock_timeout`, run `CREATE INDEX CONCURRENTLY`, and avoid DDL in peak hours.
- `VACUUM FULL` rewrites the table under an exclusive lock; prefer `pg_repack` for online reclaim.
- Row locks from `SELECT FOR UPDATE` can block writers; use `SKIP LOCKED` for queue-like workloads when appropriate.
- Monitor blocked sessions:

```sql
SELECT pid, now() - query_start AS duration, state,
       wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY duration DESC;

SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```

- Use consistent lock ordering to avoid deadlocks; analyze the deadlock log when they occur.

### Buffer cache

- `shared_buffers` holds PostgreSQL pages; the OS page cache also caches files. A miss in `shared_buffers` is not necessarily a physical read.
- Large sequential scans use a small buffer ring to avoid evicting useful pages; this is why a seq scan may not pollute the cache as much as expected.
- Check cache behavior with `pg_statio_user_tables` and the optional `pg_buffercache` extension; do not tune on hit ratio alone.

### Connections

- Pool at the application layer (this repo uses `bb8`/`psycopg` pools) or with `pgbouncer` for high concurrency. Each PostgreSQL connection is a process; more connections increase memory and contention.
- Set timeouts: `statement_timeout`, `lock_timeout`, `idle_in_transaction_session_timeout`.
- Monitor:

```sql
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state
ORDER BY count(*) DESC;
```

## Monitoring queries

```sql
-- Tables with dead tuples and maintenance history
SELECT relname, n_live_tup, n_dead_tup, last_vacuum, last_autovacuum,
       last_analyze, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Heap and index cache behavior
SELECT relname, heap_blks_hit, heap_blks_read,
       round(100.0 * heap_blks_hit / nullif(heap_blks_hit + heap_blks_read, 0), 2) AS hit_pct
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC;

-- Database-wide cache hit ratio
SELECT datname, blks_hit, blks_read,
       round(100.0 * blks_hit / nullif(blks_hit + blks_read, 0), 2) AS hit_pct
FROM pg_stat_database;

-- Table and index sizes
SELECT schemaname, relname,
       pg_size_pretty(pg_total_relation_size(format('%I.%I', schemaname, relname))) AS total_size
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(format('%I.%I', schemaname, relname)) DESC
LIMIT 20;
```

- `pg_stat_statements` must be loaded via `shared_preload_libraries`; restart is required.
- `auto_explain` can log plans for slow queries; useful when the slow query is hard to reproduce.
- Use `EXPLAIN (ANALYZE, BUFFERS)` on the exact query and parameters; a plan for different parameters can mislead.

## Output format

```
## Query Performance Analysis

**Query:** [SQL]
**Environment:** PostgreSQL version, table/index sizes, row counts, relevant settings
**Plan evidence:** [key nodes, estimates vs actual, buffers, temp, sort/hash spills]
**Root cause:** [statistics | storage/bloat | access path | join/sort | locking | configuration]

**Changes:**
1. Query rewrite: [SQL and why]
2. Index: [DDL and why]
3. Statistics/maintenance: [ANALYZE / VACUUM / extended statistics]
4. Configuration: [only if justified]

**Before/after:**
- Estimated vs actual rows: ...
- Buffers / temp blocks: ...
- Execution time: ...

**Expected impact:** [measured or estimated]
**Verification:** [commands to re-run and what to compare]
```

## Checklist

- [ ] `EXPLAIN (ANALYZE, BUFFERS)` captured before and after
- [ ] Plan read inside out; the dominant node and row-estimate errors identified
- [ ] Sequential scans on large tables justified or fixed
- [ ] Access path matches selectivity and correlation
- [ ] Index-only scans show `Heap Fetches: 0` or an explanation
- [ ] Join method and `loops` match cardinalities
- [ ] No unexpected `external merge` or `Hash Batches > 1`
- [ ] `WHERE` predicates are sargable; no functions/casts on indexed columns
- [ ] Indexes cover FK and hot `WHERE`/`ORDER BY`/`JOIN` columns
- [ ] Index type matches the data and operators (`B-tree`, `GIN`, `GiST`, `BRIN`)
- [ ] Composite index column order matches equality, then range/sort
- [ ] Partial/expression/covering indexes match query predicates exactly
- [ ] Unused, duplicate, and bloated indexes reviewed
- [ ] Statistics fresh; extended statistics where predicates are correlated
- [ ] Autovacuum healthy; dead tuples and table/index bloat under control
- [ ] `fillfactor` considered for update-heavy tables and HOT updates
- [ ] Long transactions, lock waits, and idle-in-transaction sessions checked
- [ ] `work_mem`/`maintenance_work_mem` sized for the workload and concurrency
- [ ] WAL/checkpoint settings reviewed for write latency
- [ ] Parallel workers planned and launched as expected
- [ ] Pagination is keyset-based for large offsets
- [ ] Connections pooled; `pg_stat_statements` monitored
- [ ] Queries parameterized; dynamic `ORDER BY` allowlisted
- [ ] Changes verified by measurement, not assumption
