# Planner: Execution Stages, Statistics, and Access Methods

## Description

How PostgreSQL turns SQL into a plan and why it chooses the one it does. The planner estimates row counts from statistics, assigns costs from settings, and picks access paths and join methods. Bad estimates are the most common cause of bad plans; stale statistics, correlated predicates, and functions on columns are the usual culprits. Understand the stages and the statistics before forcing a plan shape.

## When to use

- `EXPLAIN` estimates differ from actual rows by 10x or more.
- The planner picks a nested loop, hash join, or seq scan that looks wrong.
- After a bulk load, migration, or large data-distribution change.
- A prepared statement is fast for one parameter set and slow for another.
- Extended statistics, expression indexes, or `default_statistics_target` are being considered.

## Stages

1. **Parse** — syntax and catalog lookup; produces a parse tree.
2. **Rewrite** — expands views, rules, and row-level security.
3. **Plan** — estimates cardinalities and costs, chooses access paths, join order, and join methods.
4. **Execute** — runs the plan; `EXPLAIN ANALYZE` measures this stage.

The **simple query protocol** sends SQL as text. The **extended protocol** separates parse/bind/execute, supports parameters, and enables prepared statements. Prepared statements may use a **generic plan** (parameter-independent) or a **custom plan** (built for the actual parameter values). A generic plan can be bad for skewed parameters.

## Checks

```sql
-- Full evidence, including settings
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS)
SELECT ...;

-- Relation-level statistics
SELECT reltuples, relpages, relallvisible
FROM pg_class
WHERE relname = 'orders';

-- Column statistics
SELECT attname, n_distinct, null_frac, avg_width, correlation
FROM pg_stats
WHERE schemaname = 'public' AND tablename = 'orders'
ORDER BY attname;

-- MCV and histogram for one column
SELECT most_common_vals, most_common_freqs, histogram_bounds
FROM pg_stats
WHERE schemaname = 'public' AND tablename = 'orders'
  AND attname = 'status';
```

| Field | Meaning |
|---|---|
| `reltuples` | Estimated row count (updated by `ANALYZE` and some DDL) |
| `relpages` | Relation size in pages; drives sequential-scan cost |
| `relallvisible` | Pages marked all-visible; used for index-only scan cost |
| `n_distinct` | Distinct values; negative means a fraction of rows |
| `null_frac` | Fraction of NULLs |
| `correlation` | Physical vs logical order, in `[-1, 1]`; near ±1 favors index scans |
| `avg_width` | Average row width; drives sort/hash memory estimates |

`ANALYZE` samples rows and pages; the sample size scales with `default_statistics_target` (default 100). Statistics are estimates, not exact counts; an error within an order of magnitude is normal and usually enough.

## Code examples

### Compare estimates with actuals

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM orders
WHERE user_id = 42 AND status = 'pending';
```

Look for nodes where `rows` and `actual rows` differ by 10x or more. Multiply actual rows by `loops` on inner nodes to get the global count. The node with the largest row-estimate error is usually the cause of a bad join method or join order.

### Refresh statistics after a load

```sql
-- Bulk load
COPY orders FROM '/data/orders.csv' WITH (FORMAT csv);

ANALYZE orders;
```

`ANALYZE` is cheap compared with `VACUUM` and should run after every bulk load or large data change. Autovacuum analyzes too, but its threshold may be too high for a fast-growing table.

### Raise statistics detail for a skewed column

```sql
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
ANALYZE orders;
```

Raise `STATISTICS` only for columns that are skewed or frequently filtered. A high target on every column increases `ANALYZE` time and catalog size.

### Extended statistics for correlated predicates

```sql
-- The planner assumes predicates are independent
CREATE STATISTICS orders_user_status (dependencies)
ON user_id, status FROM orders;

CREATE STATISTICS orders_user_status_nd (ndistinct)
ON user_id, status FROM orders;

CREATE STATISTICS orders_user_status_mcv (mcv)
ON user_id, status FROM orders;

ANALYZE orders;
```

Use extended statistics when `WHERE a = ... AND b = ...` is consistently underestimated because the columns are correlated.

### Expression statistics

```sql
-- The query must use the same expression form
CREATE INDEX idx_users_lower_email ON users (lower(email));

CREATE STATISTICS users_lower_email_stats ON (lower(email)) FROM users;
ANALYZE users;
```

A function on an indexed column defeats column statistics and ordinary indexes. An expression index plus expression statistics is the fix; rewriting to a sargable predicate is often better.

### Inspect and adjust the cost model

```sql
SELECT name, setting, unit
FROM pg_settings
WHERE name IN (
  'seq_page_cost', 'random_page_cost', 'cpu_tuple_cost',
  'cpu_operator_cost', 'effective_cache_size',
  'work_mem', 'hash_mem_multiplier'
);
```

- `seq_page_cost` (1.0) is the reference unit.
- `random_page_cost` (4.0) should be lower on SSDs (for example 1.1–2.0).
- `effective_cache_size` is a planner hint, not an allocation.
- `hash_mem_multiplier` (1.0) multiplies `work_mem` for hash operations.

Lowering `random_page_cost` makes index access look cheaper; lowering it too much produces many random index scans on data that would be better scanned sequentially.

### Diagnose generic vs custom plans

```sql
-- Force a generic plan to reproduce a prepared-statement problem
SET plan_cache_mode = force_generic_plan;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE user_id = $1;

RESET plan_cache_mode;
```

If the generic plan is much worse, the parameter distribution is skewed. Options: `plan_cache_mode = auto` (default) or `force_custom_plan`, or rewrite the query so one plan works for all parameters.

## Anti-patterns

1. Forcing plans with `enable_seqscan = off`/`enable_nestloop = off` instead of fixing statistics or indexes.
2. Leaving statistics stale after a bulk load or migration.
3. Assuming predicates are independent when the columns are correlated.
4. Wrapping indexed columns in functions or casts in `WHERE`.
5. Raising `default_statistics_target` globally instead of per column.
6. Trusting a plan captured with different parameters or a different data distribution.
7. Comparing `EXPLAIN` output from cold and warm runs without noting it.
8. Ignoring `width`, which drives sort/hash memory and spill.
9. Using `SELECT *` in a query whose plan depends on row width.
10. Fixing a generic-plan problem by disabling prepared statements everywhere.

## Checklist

- Is the plan from `EXPLAIN (ANALYZE, BUFFERS)` with the real parameters?
- Which node has the largest row-estimate error or actual time?
- Are `reltuples`, `relpages`, and `relallvisible` current?
- Are column statistics fresh (`ANALYZE` after changes)?
- Is the column skewed? Is `STATISTICS` high enough?
- Are predicates correlated? Are extended statistics defined?
- Are predicates sargable, with no functions/casts on indexed columns?
- Do cost settings match the hardware (especially `random_page_cost`)?
- Is the plan generic or custom, and does the parameter distribution explain it?
- Were changes verified by re-running the same query and comparing estimates, buffers, and time?

## Related

- `scans-joins.md` for the access paths and join methods the planner chooses.
- `index-types.md` for which index can serve which predicate.
- `data.md` and `tables.md` for why `relpages` and visibility affect cost.
- `parallel.md` for parallel plan choices and worker limits.
