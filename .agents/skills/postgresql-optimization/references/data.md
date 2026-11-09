# Data Organization: Pages and Tuples

## Description

How PostgreSQL stores rows on disk, which explains most "the index exists but the query is still slow" and "the table is bigger than its data" cases. Tables are **heaps**: unordered collections of 8 KB pages. Each page holds item pointers, free space, and tuple versions. An `UPDATE` writes a new version and marks the old one dead; it never overwrites in place. Indexes reference tuples by TID (`page, offset`), and TOAST moves large values out of line.

## When to use

- A row or table is much larger than the data suggests.
- `UPDATE`/`DELETE` behavior, dead tuples, or bloat is suspected.
- An index-only scan still fetches from the heap.
- Large `text`, `jsonb`, or `bytea` values increase I/O.
- A row changes `ctid` after an update and an application depends on it.
- Designing `fillfactor`, partitioning, or a storage layout.

## Checks

```sql
-- Enable page inspection
CREATE EXTENSION IF NOT EXISTS pageinspect;

-- Page header: free space boundaries
SELECT lower, upper, special, pagesize
FROM page_header(get_raw_page('accounts', 0));

-- Tuple pointers and headers on page 0
SELECT lp, lp_off, lp_flags, lp_len, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('accounts', 0))
ORDER BY lp;

-- Row size and TOAST behavior
SELECT pg_column_size(payload) AS stored_bytes,
       length(payload::text) AS logical_bytes
FROM accounts
LIMIT 5;
```

Inside a page:

| Part | Contents |
|---|---|
| Page header | Checksum, flags, free-space boundaries (`pd_lower`, `pd_upper`, `pd_special`) |
| Item pointers | 4 bytes each: tuple offset, length, status bits |
| Free space | One contiguous chunk, tracked by the free space map |
| Tuples | Row versions, growing down from the special space |
| Special space | Index-specific metadata (empty for heap pages) |

A tuple header contains at least `xmin` (creating transaction), `xmax` (deleting/locking transaction), `infomask` (flags), `ctid` (next version of the same row), plus a null bitmap and alignment padding. The header is at least 23 bytes and often larger, so a "narrow" table can spend most of its space on metadata.

## Code examples

### A `ctid` changes when a row is updated

```sql
SELECT ctid, id, amount FROM accounts WHERE id = 1;
-- (0,1) | 1 | 100.00

UPDATE accounts SET amount = 150.00 WHERE id = 1;

SELECT ctid, id, amount FROM accounts WHERE id = 1;
-- (0,2) | 1 | 150.00
```

The old version at `(0,1)` is marked dead and remains until vacuumed. Index entries may still point to it, so the executor follows the `ctid` chain and checks visibility. Never use `ctid` as a stable primary key.

### Field order affects row size

```sql
CREATE TABLE padding_bad (
    b1 boolean,
    i1 integer,
    b2 boolean,
    i2 integer
);

CREATE TABLE padding_good (
    i1 integer,
    i2 integer,
    b1 boolean,
    b2 boolean
);

SELECT pg_column_size('(true,1,false,2)'::padding_bad)  AS bad,
       pg_column_size('(1,2,true,false)'::padding_good) AS good;
-- bad = 40, good = 32 (header + alignment padding differ)
```

Reorder columns so fixed-width values are adjacent, or let a separate table absorb rarely used wide columns. The effect is small for narrow tables and significant for wide, hot rows.

### TOAST moves large values out of line

```sql
-- Large values are compressed and/or stored in a TOAST table
SELECT pg_column_size(payload)            AS inline_bytes,
       octet_length(payload::text)        AS logical_bytes
FROM events
WHERE id = 42;

-- Table and TOAST sizes are reported separately
SELECT pg_size_pretty(pg_relation_size('events'))       AS heap,
       pg_size_pretty(pg_total_relation_size('events')) AS total_including_toast;
```

Selecting a large TOASTed column that the application does not need still pays decompression and I/O. Avoid `SELECT *` on tables with wide `text`, `jsonb`, or `bytea` columns.

### Index entries reference tuple versions

```sql
-- After several updates, the index contains entries for old versions too
SELECT itemoffset, ctid, dead
FROM bt_page_items('accounts_pkey', 1)
ORDER BY itemoffset
LIMIT 10;
```

An index scan may return a TID whose heap tuple is dead; the executor checks visibility and ignores it. Vacuum later removes both the heap versions and the index entries. Until then, index scans can do extra heap visits.

### Inspect free space and fillfactor

```sql
SELECT lower, upper, pagesize,
       round(100.0 * (upper - lower) / pagesize, 1) AS free_pct
FROM page_header(get_raw_page('orders', 0));

SHOW fillfactor;   -- default 100 for tables
```

`fillfactor` below 100 reserves space per page for updates, enabling HOT updates when the changed columns are not indexed. It trades space for fewer index writes and less bloat.

## Anti-patterns

1. Treating `ctid` as a stable identifier; it changes on update and after vacuum.
2. Expecting `UPDATE` to modify a row in place; it creates a new version.
3. `SELECT *` on tables with large TOASTed columns.
4. Ignoring tuple-header overhead when designing very narrow tables.
5. Assuming an index entry points to a live tuple; visibility is checked per version.
6. Using `fillfactor = 100` on an update-heavy table.
7. Assuming table file size equals live data size; dead versions count until vacuumed.
8. Storing large binary objects in the heap when a separate object store is more appropriate.

## Checklist

- Do you know the page size (8 KB) and the tuple header size (>= 23 bytes)?
- Is the table update-heavy? Consider `fillfactor` and HOT.
- Are wide columns TOASTed, and does the query avoid selecting them?
- Is `ctid` used anywhere as a durable key?
- Are dead tuples and index entries being cleaned by vacuum?
- Does the row layout have avoidable padding?
- Is the table's physical size compared with `n_live_tup`, not just row count?

## Related

- `tables.md` for bloat, vacuum, HOT, and reclaiming space.
- `cache.md` for how pages move between disk and shared buffers.
- `index-types.md` for what index entries store.
- `scans-joins.md` for how scans use TIDs and visibility.
