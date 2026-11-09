# Index Types

## Description

Which index access method to use, and why. B-tree covers equality, ranges, ordering, and uniqueness. Hash is equality-only. GiST handles geometry, ranges, and nearest-neighbor. SP-GiST handles space-partitioned and text data. GIN handles containment in JSONB, arrays, full-text, and trigrams. BRIN summarizes large, physically ordered tables with a tiny index. Each has different write costs, lossiness, and operator support.

## When to use

- Adding an index for a predicate, join, or ordering.
- An existing index is not used, or is used with heavy rechecking.
- Choosing between B-tree, GIN, GiST, and BRIN for a data type.
- A JSONB, array, full-text, geometric, or range query is slow.
- An index is large, bloated, or expensive to write.
- Reviewing unused or duplicate indexes.

## Index types

| Type | Best for | Supports | Limits |
|---|---|---|---|
| **B-tree** | Equality, ranges, sorting, uniqueness | Multicolumn, `INCLUDE`, index-only, backward scan, `NULLS FIRST/LAST`, expression/partial | Not for containment, JSONB, arrays, full-text, nearest-neighbor |
| **Hash** | Equality only | Equality, bitmap scans | No ordering, uniqueness, multicolumn, `INCLUDE`, or index-only; rarely beats B-tree |
| **GiST** | Geometry, ranges, full-text, KNN with an operator class | Distance ordering, exclusion constraints (`btree_gist`) | Operator-class dependent; lossy rechecks; larger/slower for simple equality |
| **SP-GiST** | Points, text, space-partitioned data | Quadtree, k-d tree, radix tree structures | Niche; operator-class dependent |
| **GIN** | JSONB, arrays, `tsvector`, trigrams | Containment, existence, full-text search | Expensive writes; pending list; no index-only scans |
| **BRIN** | Very large append-only tables with correlated data | Min/max summaries, tiny index | Lossy; poor for random lookups; requires physical correlation |

## Code examples

### B-tree: composite, partial, expression, covering

```sql
-- Composite: equality columns first, then range/sort
CREATE INDEX idx_orders_user_date ON orders (user_id, order_date);

-- Partial: index only the rows the query touches
CREATE INDEX idx_orders_pending ON orders (created_at)
WHERE status = 'pending';

-- Expression: for computed predicates
CREATE INDEX idx_users_lower_email ON users (lower(email));

-- Covering: include output columns, not search keys
CREATE INDEX idx_orders_covering ON orders (user_id, status)
INCLUDE (total, created_at);

-- Prefix matching under a non-C collation
CREATE INDEX idx_users_email_pattern ON users (email text_pattern_ops);

-- Match ordering and null placement when the query depends on it
CREATE INDEX idx_tickets_name_bref
ON tickets (passenger_name NULLS FIRST, book_ref DESC);
```

A B-tree cannot efficiently use a later column without the leading columns. `INCLUDE` columns are not searchable; they exist to make index-only scans possible.

### Hash: equality only

```sql
CREATE INDEX idx_sessions_token ON sessions USING hash (token);
```

Hash indexes are smaller than B-trees for long equality-only keys and support bitmap scans. They do **not** provide ordering, uniqueness, multicolumn indexes, `INCLUDE`, or index-only scans. For most workloads a B-tree is at least as good; measure before choosing hash.

### GIN: JSONB, arrays, full-text, trigrams

```sql
-- JSONB containment
CREATE INDEX idx_events_payload ON events USING gin (payload jsonb_path_ops);

-- Array membership
CREATE INDEX idx_posts_tags ON posts USING gin (tags);

-- Full-text search
CREATE INDEX idx_docs_tsv ON docs USING gin (to_tsvector('english', body));

-- Trigram: leading wildcards and ILIKE (requires pg_trgm)
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_users_name_trgm ON users USING gin (name gin_trgm_ops);
```

- `jsonb_path_ops` is smaller and faster than the default `jsonb_ops` but supports fewer operators. Choose by the operators the query uses.
- GIN writes are expensive because one row can produce many index entries. `fastupdate` and its pending list trade write cost for read cost; check `gin_pending_list_limit` and clean the pending list when reads are slow.
- GIN does not support index-only scans.

### GiST: geometry, ranges, nearest neighbor

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- Exclusion constraint: no overlapping bookings for a room
ALTER TABLE bookings
ADD CONSTRAINT no_overlap
EXCLUDE USING gist (room_id WITH =, period WITH &&);

-- Nearest neighbor with a KNN operator
CREATE INDEX idx_places_location ON places USING gist (location);
SELECT * FROM places ORDER BY location <-> point '(1,2)' LIMIT 10;
```

GiST is extensible and operator-class dependent. It can return lossy results that require a heap recheck; watch `Rows Removed by Index Recheck`. `btree_gist` enables GiST indexes on ordinary scalar types for exclusion constraints and mixed multicolumn indexes.

### SP-GiST: space-partitioned structures

```sql
CREATE INDEX idx_points_location ON points USING spgist (location);
CREATE INDEX idx_names_prefix ON names USING spgist (name text_ops);
```

SP-GiST implements quadtrees, k-d trees, and radix trees. It is excellent for non-overlapping, space-partitioned data and prefix search on text, but it is niche and operator-class dependent. Measure against a B-tree or GiST for the specific query.

### BRIN: large, append-only, correlated data

```sql
CREATE INDEX idx_metrics_time_brin ON metrics USING brin (recorded_at)
WITH (pages_per_range = 32);
```

BRIN stores only summaries (usually min/max) per range of pages; the index can be orders of magnitude smaller than a B-tree. It only helps when values are physically correlated with insertion order, and it is lossy: matching ranges return all their pages for rechecking. `pages_per_range` controls granularity: smaller ranges mean more entries and less rechecking.

## Choosing

| Predicate / need | Index |
|---|---|
| `=`, `<`, `>`, `BETWEEN`, `ORDER BY`, unique | B-tree |
| `=` on a long opaque key, no ordering | Hash (rarely better than B-tree) |
| `jsonb @>`, `?`, `?|`, `?&` on document keys | GIN (`jsonb_path_ops` for `@>`) |
| Array `@>`, `&&`, `= ANY` | GIN |
| `tsvector @@ tsquery` | GIN |
| `LIKE '%x%'`, `ILIKE` | GIN trigram (`pg_trgm`) |
| Geometry, ranges, KNN distance | GiST |
| Space-partitioned points, text prefixes | SP-GiST |
| Huge append-only table, time-ordered scans | BRIN |

## Code examples: find bloated or unused indexes

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

```sql
-- Confirm index bloat / leaf density
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT * FROM pgstatindex('idx_orders_user_date');
```

## Anti-patterns

1. Using GIN for equality lookups that a B-tree handles better.
2. Using a B-tree for `jsonb @>` or array containment; it cannot serve them.
3. Using BRIN on randomly ordered data; it becomes a lossy full scan.
4. Using GiST/GIN when rechecking may dominate; inspect `Recheck Cond`.
5. Creating an index on every column "just in case"; writes and space suffer.
6. Ignoring composite column order (equality first, then range/sort).
7. Expecting an expression index to be used when the query expression differs.
8. Expecting a partial index to be used when its predicate is not implied.
9. Using `INCLUDE` columns for search or ordering.
10. Leaving unused or duplicate indexes in place.

## Checklist

- Which operators does the query use? Does the index support them?
- Is a B-tree sufficient, or is a specialized type required?
- For composite indexes, is the column order equality → range/sort?
- For partial indexes, does the query imply the predicate?
- For expression indexes, is the expression identical?
- For GIN, are write cost, pending list, and lack of index-only acceptable?
- For GiST/BRIN, is lossy rechecking acceptable?
- For BRIN, is the data physically correlated with the indexed value?
- Is the index used (`idx_scan`) and not bloated (`pgstatindex`)?
- Was the choice validated with `EXPLAIN (ANALYZE, BUFFERS)` and write benchmarks?

## Related

- `planner.md` for how the planner chooses an index.
- `scans-joins.md` for index, index-only, and bitmap scan behavior.
- `tables.md` for index bloat and rebuilds.
- `write-latency.md` for the write cost of indexes.
