# Buffer Cache

## Description

How PostgreSQL caches relation pages and why the cache may not be helping. The buffer cache (`shared_buffers`) is a shared array of 8 KB buffers, indexed by a hash table keyed on the page's file/fork/block. Each buffer tracks whether it is dirty, how often it was used, and whether it is pinned. A miss does not necessarily mean a disk read: the OS page cache may satisfy it. Tuning by hit ratio alone is misleading.

## When to use

- `EXPLAIN (ANALYZE, BUFFERS)` shows high `shared read` and low `shared hit`.
- The working set does not fit in memory after a size or traffic change.
- Deciding how much RAM to give `shared_buffers` versus the OS cache.
- A large sequential scan seems to evict useful pages.
- Investigating the effect of `pg_prewarm` after a restart.
- A query performs worse after a deployment despite unchanged SQL.

## Checks

```sql
-- Cache contents for one relation (requires pg_buffercache)
CREATE EXTENSION IF NOT EXISTS pg_buffercache;

SELECT count(*)                              AS buffers,
       count(*) FILTER (WHERE isdirty)       AS dirty,
       round(avg(usagecount), 2)             AS avg_usage
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE c.relname = 'orders';

-- Per-table cache behavior
SELECT relname, heap_blks_hit, heap_blks_read,
       round(100.0 * heap_blks_hit / nullif(heap_blks_hit + heap_blks_read, 0), 2) AS hit_pct
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC
LIMIT 20;

-- Database-wide
SELECT datname, blks_hit, blks_read,
       round(100.0 * blks_hit / nullif(blks_hit + blks_read, 0), 2) AS hit_pct
FROM pg_stat_database;

SHOW shared_buffers;
SHOW effective_cache_size;
```

`EXPLAIN (ANALYZE, BUFFERS)` distinguishes:

| Field | Meaning |
|---|---|
| `shared hit` | Page found in `shared_buffers` |
| `shared read` | Page not in `shared_buffers`; may still hit the OS cache |
| `temp read/written` | Spilled sort/hash blocks |
| `local hit/read` | Temporary/local buffers |

## Code examples

### Buffer cache is not the whole story

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE user_id = 42;
```

A high `shared read` count means pages were not in PostgreSQL's cache. They may still have been served from the OS page cache, so the query can be fast while `shared read` is high. Correlate with actual time and disk counters; do not tune `shared_buffers` from the hit ratio alone.

### Size `shared_buffers` deliberately

```sql
-- Check current value and unit
SHOW shared_buffers;

-- Example: set in postgresql.conf (requires restart)
-- shared_buffers = 4GB
```

Start around 25% of RAM and tune with measurements. More is not always better: PostgreSQL also relies on the OS page cache, and an oversized shared cache can increase checkpoint and eviction work. `effective_cache_size` is a **planner hint** for the combined PostgreSQL + OS cache, not an allocation.

### Cache warming

```sql
-- Load a relation into shared buffers (requires pg_prewarm)
CREATE EXTENSION IF NOT EXISTS pg_prewarm;
SELECT pg_prewarm('orders');
```

Use after a restart or for a known hot table. Do not prewarm everything: it competes with the working set and delays serving real queries.

### Large sequential scans use a ring

A large sequential scan does not stream the whole table through the cache at the expense of everything else. PostgreSQL uses a small **buffer ring**: the scan reuses a limited set of buffers instead of filling the cache with pages it will not revisit. This is why a seq scan may not pollute the cache as much as its size suggests, and why its pages may not remain cached afterward.

### Find which relation a buffer belongs to

```sql
SELECT c.relname, b.relblocknumber, b.isdirty, b.usagecount
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE c.relname = 'orders'
ORDER BY b.relblocknumber
LIMIT 20;
```

`usagecount` reflects the clock-sweep usage counter; higher means more recently reused. `pinning_backends` shows how many backends currently pin the page.

### Cache misses caused by the plan, not the size

A nested loop with a high `loops` count can read the same inner pages repeatedly. A hash join or a better index may reduce the number of buffer accesses more than adding memory.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...
FROM small s
JOIN large l ON l.key = s.key;
```

Check `loops`, `shared hit/read` per node, and whether a `Memoize` node is caching repeated inner lookups.

## Anti-patterns

1. Tuning `shared_buffers` from the database hit ratio alone.
2. Setting `effective_cache_size` to a real allocation instead of an estimate of OS + PostgreSQL cache.
3. Assuming every `shared read` is a physical disk read.
4. Prewarming everything after a restart.
5. Expecting the cache to fix a bad plan, a missing index, or bloat.
6. Giving PostgreSQL nearly all RAM and starving the OS page cache.
7. Ignoring `temp` buffers from sorts/hashes when diagnosing I/O.
8. Measuring cache behavior on a cold or noisy instance and generalizing.
9. Forgetting that a large seq scan uses a ring and will not warm the cache.
10. Comparing `EXPLAIN` runs taken with different cache states.

## Checklist

- Is the working set larger than `shared_buffers` plus the OS cache?
- Are `shared read` and actual time both high, or only `shared read`?
- Is the plan reading pages repeatedly (nested loop with high `loops`)?
- Is `shared_buffers` sized from measurement, not a rule of thumb alone?
- Is `effective_cache_size` a realistic estimate for the host?
- Are sorts/hashes spilling (`temp read/written`)?
- Was the effect of caching changes measured on a warm instance?
- Does the OS have enough free memory for its page cache?

## Related

- `data.md` for pages and how they are laid out.
- `tables.md` for bloat, which inflates the working set.
- `write-latency.md` for dirty pages and checkpoints.
- `scans-joins.md` for plans that touch fewer or more buffers.
