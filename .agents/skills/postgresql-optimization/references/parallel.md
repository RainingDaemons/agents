# Parallel Plans

## Description

When PostgreSQL splits a query across worker processes, and why it sometimes does not help. The leader spawns workers, each runs the parallel part of the plan, and `Gather` (or `Gather Merge`) collects the results. Starting processes and transferring tuples is not free: parallelism pays off for large scans and aggregations, not for small OLTP queries, write-heavy work, or plans with a small `LIMIT`. Not every node can run in parallel.

## When to use

- A large scan or aggregation underuses the CPU.
- `Workers Planned` is greater than `Workers Launched`.
- A parallel plan is slower than the serial equivalent.
- Tuning `max_parallel_workers_per_gather`, `max_parallel_workers`, `max_worker_processes`, or `parallel_leader_participation`.
- Deciding whether to force `parallel_workers` on a large table.
- Investigating why a parallel plan does not appear.

## Plan structure

```text
Finalize Aggregate
  -> Gather
       Workers Planned: 2
       -> Partial Aggregate
            -> Parallel Seq Scan on bookings
```

Everything below `Gather` is the **parallel part**, executed by each worker (and possibly the leader). `Gather` and everything above it is the **sequential part**, executed by the leader alone.

| Node | Role |
|---|---|
| `Gather` | Collects worker tuples (order not preserved) |
| `Gather Merge` | Collects worker tuples in sorted order |
| `Parallel Seq Scan` | Workers read disjoint page ranges |
| `Parallel Index Scan` / `Parallel Bitmap Heap Scan` | Parallel index-driven reads |
| `Partial Aggregate` | Per-worker aggregation, finalized above `Gather` |

## Settings

| Setting | PostgreSQL 14 default | Effect |
|---|---|---|
| `max_parallel_workers_per_gather` | 2 | Workers per plan |
| `max_parallel_workers` | 8 | Global parallel worker slots |
| `max_worker_processes` | 8 | Global background worker slots |
| `min_parallel_table_scan_size` | 8 MB | Table size gate for parallel scans |
| `min_parallel_index_scan_size` | 512 kB | Index size gate |
| `parallel_setup_cost` | 1000 | Planner cost of starting workers |
| `parallel_tuple_cost` | 0.1 | Planner cost per transferred tuple |
| `parallel_leader_participation` | on | Whether the leader also runs the parallel part |
| `force_parallel_mode` | off | Test-only: force parallel plans |

## Code examples

### Read a parallel plan

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM bookings;
```

```text
Finalize Aggregate
  -> Gather
       Workers Planned: 2
       Workers Launched: 2
       -> Partial Aggregate
            -> Parallel Seq Scan on bookings
```

`Workers Launched < Workers Planned` means the plan wanted workers but could not get them: global limits, contention, or a busy server. Check `max_parallel_workers` and `max_worker_processes`, plus overall load.

### Compare serial and parallel

```sql
-- Parallel
EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM bookings;

-- Serial baseline
SET max_parallel_workers_per_gather = 0;
EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM bookings;
RESET max_parallel_workers_per_gather;
```

Compare actual time, buffers, and CPU. Parallelism often reduces wall time while increasing total CPU and buffer touches. If the serial plan is faster on a small table, let the planner choose serial; do not force parallelism.

### A parallel plan that does not help

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM bookings ORDER BY book_date LIMIT 10;
```

A small `LIMIT` can make workers do more work than the leader needs: the leader may stop early, but the workers still scan and transfer rows. The planner accounts for `parallel_tuple_cost`, but a bad estimate or a large `LIMIT` can still make parallelism a loss. Compare with the serial plan.

### Force workers on a large table (deliberately)

```sql
ALTER TABLE bookings SET (parallel_workers = 4);

-- Or lower the size gate for a specific experiment
SET min_parallel_table_scan_size = 0;
EXPLAIN (ANALYZE) SELECT count(*) FROM bookings;
RESET min_parallel_table_scan_size;
```

`parallel_workers` is a per-table reloption; use it when the planner underestimates the benefit, not as a global default. Lowering `min_parallel_table_scan_size` globally creates parallel plans for small tables, where overhead wins.

### Leader participation

```sql
SET parallel_leader_participation = off;
EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM bookings;
RESET parallel_leader_participation;
```

With participation on, the leader also scans; this is usually good. Turning it off can help latency-sensitive queries where the leader should focus on collecting results. Measure.

### Parallel index scans

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM events WHERE created_at >= now() - interval '1 day';
```

A `Parallel Index Only Scan` or `Parallel Bitmap Heap Scan` can appear when the index is large enough and the predicate matches enough rows. Parallel index scans still pay heap access and transfer costs; check buffers and compare with the serial plan.

## Limitations

- **Not all nodes are parallel-aware.** Some operations run only in the leader.
- **Writes are not parallelized** the same way; `INSERT`, `UPDATE`, and `DELETE` do not get parallel workers in PostgreSQL 14.
- **Parallel-safe functions are required.** User-defined functions must be declared `PARALLEL SAFE`; otherwise they force a serial plan.
- **Snapshot and locking semantics** can restrict parallelism for cursors, some `FOR UPDATE`, and certain isolation behaviors.
- **Worker slots are global.** A few concurrent large queries can consume all workers; later queries silently lose parallelism.
- **Memory multiplies.** Each worker has its own `work_mem` for sorts and hashes; more workers can mean more total memory and more contention.
- **Transfer overhead.** `Gather` moves tuples from workers to the leader; wide rows and high row counts make this expensive.

## Anti-patterns

1. Setting `max_parallel_workers_per_gather` very high as a default.
2. Parallelizing small OLTP queries where setup cost dominates.
3. Expecting writes to be parallelized in PostgreSQL 14.
4. Forcing `parallel_workers` everywhere instead of fixing estimates.
5. Ignoring `Workers Launched < Workers Planned`.
6. Using `force_parallel_mode` in production.
7. Parallelizing queries with a small `LIMIT` that the leader does not need.
8. Raising `work_mem` and worker count together without accounting for memory multiplication.
9. Comparing parallel and serial plans on different cache states.
10. Assuming more workers always means lower latency.

## Checklist

- Is the query large enough for parallel overhead to be paid back?
- Is the dominant node actually in the parallel part of the plan?
- Are planned workers actually launched?
- Do global worker limits or concurrent queries explain missing workers?
- Is the serial plan faster? If so, do not force parallelism.
- Is the `LIMIT` large enough to justify scanning in parallel?
- Are functions `PARALLEL SAFE` where needed?
- Is per-worker `work_mem` accounted for in total memory?
- Does `Gather` transfer enough wide rows to offset the gain?
- Was the change measured with `EXPLAIN (ANALYZE, BUFFERS)` on the same data?

## Related

- `planner.md` for cost estimation and statistics behind the choice.
- `scans-joins.md` for the scans and joins that run in parallel.
- `cache.md` for the extra buffer and I/O behavior of workers.
- `locks.md` for contention that prevents workers from running.
