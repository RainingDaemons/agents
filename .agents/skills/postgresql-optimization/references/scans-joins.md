# Scans, Joins, and Sorts

## Description

How PostgreSQL reads rows and combines them, and how to tell whether the chosen method fits the data. Access paths trade sequential I/O against random heap visits; join methods trade memory against repeated lookups; sorts trade memory against disk. The right choice depends on selectivity, physical correlation, cardinality, and available `work_mem` — not on which node "sounds faster".

## When to use

- A plan's dominant node is a scan, join, or sort.
- A sequential scan looks wrong on a large table.
- An index scan does many heap fetches.
- A nested loop has high `loops`.
- A hash join spills to disk or a sort uses `external merge`.
- Choosing between a hash join and a merge join.

## Access paths

| Node | How it works | Best when | Watch for |
|---|---|---|---|
| `Seq Scan` | Reads all table pages | Most rows needed, small table, no usable index | Bloat inflates `relpages`; filters discard many rows |
| `Index Scan` | Walks the index, fetches heap TIDs one by one | Selective predicate, high correlation | Random heap I/O; many `shared read`; low correlation |
| `Index Only Scan` | Returns values from the index | Covering index + visibility map | `Heap Fetches > 0` means vacuum is behind |
| `Bitmap Index Scan` + `Bitmap Heap Scan` | Collects TIDs, sorts by page, reads pages once | Medium selectivity, low correlation | Lossy bitmap: `Recheck Cond`, `Rows Removed by Index Recheck` |
| `Tid Scan` | Fetches known TIDs | `WHERE ctid = ...` | Rare outside maintenance |

## Join methods

| Node | How it works | Best when | Watch for |
|---|---|---|---|
| `Nested Loop` | For each outer row, probes the inner side | Small outer set, indexed inner side | High `loops`; missing inner index; underestimated outer cardinality |
| `Hash Join` + `Hash` | Builds a hash table on the inner side | Large equality join | `Hash Batches > 1`; `Memory Usage` near `work_mem * hash_mem_multiplier`; wide rows |
| `Merge Join` | Merges two sorted inputs | Large equi-join, inputs already sorted, inequality joins | Extra `Sort` nodes; `external merge` spills |

`Materialize` caches an inner result for reuse; `Memoize` (v14+) caches parameterized inner results for repeated nested-loop lookups. Both trade memory for fewer child executions.

## Sorts

| Sort method | Meaning |
|---|---|
| `quicksort` | In-memory sort within `work_mem` |
| `top-N heapsort` | Only the first `N` rows needed (`ORDER BY ... LIMIT`) |
| `external merge` | Sort spilled to disk because it exceeded `work_mem` |

`work_mem` is per sort/hash operation, not per connection. A complex query can allocate it several times, and concurrency multiplies total memory. An index in the right order can remove the sort entirely.

## Code examples

### Seq scan is not automatically a problem

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM orders WHERE total > 0;
```

If the predicate matches most of the table, a sequential scan is cheaper than walking an index and then visiting nearly every heap page. Check `Rows Removed by Filter`: if almost every row survives, the scan is doing the right thing.

### Index scan with poor correlation

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM events WHERE created_at >= now() - interval '1 day';
```

If `correlation` on `created_at` is near 0, heap accesses jump around and `shared read` grows. A bitmap scan can be better for medium selectivity:

```sql
-- Encourage a bitmap plan to compare (diagnostic only)
SET enable_indexscan = off;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM events WHERE created_at >= now() - interval '1 day';
RESET enable_indexscan;
```

Do not leave `enable_*` toggles on; use them to compare, then fix the underlying cause (statistics, index, `VACUUM`, or `random_page_cost`).

### Index-only scan needs the visibility map

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT user_id, status FROM orders WHERE user_id = 42;
```

```text
Index Only Scan using orders_user_status_idx ...
  Heap Fetches: 0
```

`Heap Fetches > 0` means some pages are not marked all-visible. `VACUUM` (and autovacuum keeping up) is the fix, not a new index.

### Bitmap scan and lossy rechecks

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM bookings WHERE total_amount = 48500.00;
```

```text
Bitmap Heap Scan on bookings
  Recheck Cond: (total_amount = 48500.00)
  -> Bitmap Index Scan on bookings_total_amount_idx
```

The bitmap is built in `work_mem` (4 MB by default per operation). If it becomes lossy, the plan shows `Rows Removed by Index Recheck`. Increase `work_mem` for that query or reduce the number of matching pages; do not assume the recheck is free.

### Nested loop with high loops

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE u.created_at >= '2024-01-01';
```

If the outer side is estimated as 10 rows but actually returns 10,000, the nested loop runs 1,000x more inner probes than planned. Fix the estimate (`ANALYZE`, extended statistics) or provide an index on the inner join column; do not force a hash join by disabling nested loops.

### Hash join spill

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT b.book_ref, t.ticket_no
FROM bookings b
JOIN tickets t ON t.book_ref = b.book_ref;
```

```text
Hash
  Buckets: 4194304  Batches: 4  Memory Usage: 145986kB
```

`Batches > 1` means the hash table did not fit in `work_mem * hash_mem_multiplier` and was partitioned. Increase `work_mem` for the session/query, or reduce row width by selecting fewer columns. Avoid `SELECT *` in joins whose hash table is near the limit.

### Merge join and extra sorts

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM flights f
JOIN airports_data a ON f.departure_airport = a.airport_code
ORDER BY a.airport_code;
```

A merge join is attractive when both inputs are already sorted (indexes in matching order) or the query needs the sorted output anyway. If the plan adds a `Sort` with `external merge`, it is trading a hash table for disk I/O; compare the alternatives.

### Sort spill

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM events ORDER BY created_at DESC LIMIT 1000;
```

If `Sort Method: external merge`, either raise `work_mem` for the query or use an index matching `ORDER BY`. For `LIMIT`, a top-N heapsort or an index scan can avoid sorting the whole set.

## Anti-patterns

1. Treating every `Seq Scan` as a bug; on a low-selectivity predicate it is correct.
2. Adding an index without checking `correlation` and heap-fetch cost.
3. Leaving `enable_*` settings off permanently to force a plan.
4. Assuming `Heap Fetches: 0` without running `VACUUM`.
5. `SELECT *` in joins and sorts, inflating width and memory.
6. Raising `work_mem` globally instead of for the queries that need it.
7. Forcing a hash join when the real problem is a 100x cardinality error.
8. Ignoring `loops` when comparing estimated and actual rows.
9. Blaming the join method when the sort or scan below it dominates.
10. Comparing plans with different parameters or data distribution.

## Checklist

- Is the dominant node a scan, join, sort, or aggregate?
- What is the selectivity, and does the access path match it?
- What is `correlation` for the indexed column?
- Does an index-only scan show `Heap Fetches: 0`?
- Is the bitmap lossy (`Recheck Cond` / rechecked rows)?
- Are actual rows per loop close to the estimate?
- Does a hash join use one batch and fit in `work_mem`?
- Is a sort spilling (`external merge`), and can an index remove it?
- Is `work_mem` sized for the query and for concurrency?
- Were alternatives compared with measured `EXPLAIN (ANALYZE, BUFFERS)` output?

## Related

- `planner.md` for statistics and cost decisions.
- `index-types.md` for which index can serve the predicate.
- `cache.md` for interpreting buffers and cache misses.
- `parallel.md` for parallel variants of these nodes.
