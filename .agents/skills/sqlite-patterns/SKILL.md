---
name: sqlite-patterns
description: SQLite schema and runtime patterns - PRAGMA configuration, type affinity and STRICT tables, keys, constraints, storage of dates/booleans/money/UUIDs, JSON, FTS5, generated columns, UPSERT, RETURNING, transactions, connection rules, and indexing. Use when writing or reviewing SQLite DDL/DML, configuring PRAGMAs, or designing an embedded database schema.
license: MIT
metadata:
  author: rainingdaemons/agents
  version: "1.0"
  source: Generated from the SQLite documentation and the SQLite/Zipvfs coding style. Every example and default quoted here was executed against SQLite 3.50.4.
---

# SQLite Patterns

How to design, write, and review SQLite schemas and queries so they stay correct and predictable as an
application grows. This skill owns **engine-level SQLite**: PRAGMAs, affinity and STRICT, key choice,
constraints, storage conventions for scalars, SQLite-specific SQL features, connection/transaction rules,
and design-level indexing.

It is **host-agnostic**: the rules are SQL- and storage-level, so they apply to the `sqlite3` CLI, Python,
Rust, C, mobile, and WASM hosts. Where a driver changes a default, the text says so and gives the probe
that settles it.

- Engine-agnostic schema shape (normalization, relationships, migration strategy) → `database-design`
- Query plans, index deep-tuning, pagination → `postgresql-optimization` for PostgreSQL; for SQLite see
  "Indexing" and "Maintenance" below
- SQL injection, secret handling, uploads → `security-review`
- This repository's own database conventions (if any) → those documents, which win over anything here

> **Precedence.** **MUST** = correctness or safety; ignoring it is a defect. **SHOULD** = strong default;
> an existing project convention that conflicts wins, provided it is applied consistently. **PREFERENCE**
> = style; follow the repository.

## Step 0 - probe the environment before recommending anything

Never assume a version or a compiled-in feature. Run these first; they are the cheapest reproducibility
guarantee this skill offers.

```sql
SELECT sqlite_version(), sqlite_source_id();
SELECT compile_options FROM pragma_compile_options WHERE compile_options LIKE 'ENABLE_%';
-- Look for ENABLE_FTS5, ENABLE_MATH_FUNCTIONS, ENABLE_JSON*; a missing feature is a hard constraint,
-- not a style choice. Also check for OMIT_* options (OMIT_FOREIGN_KEY, OMIT_JSON, OMIT_FTS5...).
```

Driver defaults differ from the C library defaults. Two that matter in practice:

- **`busy_timeout`**: the C library defaults to `0`. Python's `sqlite3` defaults it to `5000`ms
  (`timeout=`). Read `PRAGMA busy_timeout` instead of assuming either value.
- **`:memory:` databases cannot use WAL** - `PRAGMA journal_mode = WAL` returns `memory`. A test suite
  running on `:memory:` therefore never exercises the WAL/`SQLITE_BUSY` behaviour of production.

`references/verify.sql` packages every probe referenced in this document into one runnable script.

## PRAGMA configuration

PRAGMAs are either persisted in the database file (set once) or per-connection state (must be re-applied
on **every** new connection, including pooled ones). Getting that split wrong is the most common SQLite
misconfiguration.

```sql
-- Per-database: stored in the file, survives reconnects
PRAGMA journal_mode = WAL;          -- returns the resulting mode; run once per file
PRAGMA page_size = 4096;            -- only takes effect on a new file, or after VACUUM
PRAGMA auto_vacuum = INCREMENTAL;   -- same: needs VACUUM on an existing file
PRAGMA encoding = 'UTF-8';          -- before the first table is created

-- Per-connection: run on EVERY connection, right after opening it
PRAGMA foreign_keys = ON;
PRAGMA busy_timeout = 5000;
PRAGMA synchronous = NORMAL;
PRAGMA cache_size = -65536;          -- negative = KiB, so -65536 = 64 MiB
PRAGMA journal_size_limit = 67108864; -- 64 MiB cap on the WAL after checkpointing
PRAGMA temp_store = MEMORY;
```

```sql
-- WRONG: opening a connection and issuing queries
-- Defaults are: journal_mode=delete, foreign_keys=OFF, busy_timeout=0 (C library),
-- synchronous=FULL, cache_size=-2000 (2 MiB), temp_store=file. Foreign keys are then
-- parsed but NOT enforced, and every lock conflict fails immediately.
```

Classification details that the defaults hide:

| PRAGMA | Scope | Notes |
| --- | --- | --- |
| `journal_mode` | file | WAL persists until changed; `:memory:` cannot use WAL |
| `page_size`, `auto_vacuum`, `encoding` | file | Effective value only changes on a new file or after `VACUUM` |
| `foreign_keys` | connection | OFF by default; **silently a no-op inside an open transaction** |
| `busy_timeout` | connection | C default `0`, drivers may raise it |
| `synchronous` | connection | Default FULL |
| `cache_size` | connection | Negative values are KiB |
| `journal_size_limit` | connection | Documented per-database, but it resets to `-1` on a new connection in tested builds - always set it |
| `temp_store` | connection | `MEMORY` speeds temp b-trees, costs RAM |

**MUST**: apply the whole per-connection block immediately after connecting, **outside any transaction**,
and verify - `PRAGMA foreign_keys;` must return `1`. Inside a transaction the pragma is silently ignored.

### WAL mode

WAL gives one writer concurrent with many readers, readers that never block each other, and no
rollback-journal recovery work to redo when a connection opens after a crash.

```sql
PRAGMA journal_mode = WAL;          -- verify the returned value is 'wal'
PRAGMA wal_checkpoint(PASSIVE);     -- non-blocking; automatic checkpoints do this
PRAGMA wal_checkpoint(TRUNCATE);    -- on shutdown: checkpoint and shrink the -wal file
```

Constraints and consequences:

- Requires a **local filesystem** with working shared memory. Network filesystems (NFS, SMB, CIFS) and
  some container overlay setups are unsafe with WAL - staying on `DELETE` there is correct, not lazy.
- One writer at a time. Concurrency is *readers vs writer*, not writer vs writer.
- A long-lived read transaction prevents checkpointing, so the `-wal` file grows without bound. Keep read
  transactions short in long-running services.

### Durability, not just speed

`synchronous = NORMAL` is safe against **corruption** but not against **losing the most recent
transactions** on power loss or OS crash. Choose deliberately:

- Default to `NORMAL` with WAL for throughput.
- Use `synchronous = FULL` when a committed transaction must survive a power cut (payments, audit).

### Optimizer and integrity PRAGMAs

```sql
PRAGMA optimize;                    -- before closing: refresh stats only where useful (cheap)
PRAGMA optimize(0x02);              -- ANALYZE-style limits, same idea
PRAGMA analysis_limit = 400;        -- bounds ANALYZE work on large databases
PRAGMA quick_check;                 -- fast structural check; returns 'ok'
PRAGMA integrity_check(1);          -- deeper check, cap output rows
PRAGMA trusted_schema = OFF;        -- recommended by SQLite: schema objects see no app-defined functions
```

SQLite's own documentation encourages `trusted_schema = OFF` on every connection: it stops views,
triggers, CHECK constraints, DEFAULT clauses, generated columns, and partial/expression indexes from
running application-defined functions and virtual tables. The only cost is that a schema which *relies*
on those functions stops working, so check before enabling it on an existing database.

## Type affinity

SQLite types by **affinity**, not by declaration. The declared type only selects the affinity; the value
is then converted at write time.

| Affinity | Declared type contains | Effect on write |
| --- | --- | --- |
| INTEGER | `INT` | Integers kept; lossless numeric text converted (`'100'` -> 100); `1.5` stays REAL, `'abc'` stays TEXT |
| TEXT | `CHAR`, `CLOB`, `TEXT` | Numbers converted to text |
| BLOB | `BLOB`, or no type at all | Stored exactly, no conversion |
| REAL | `REAL`, `FLOA`, `DOUB` | Stored as float (integers may remain integers) |
| NUMERIC | anything else (`NUMERIC`, `DECIMAL`, `BOOLEAN`, `DATE`, `DATETIME`) | Integer/real if lossless, else text |

Verified consequences of affinity (they surprise people):

```sql
CREATE TABLE aff(x INTEGER);
INSERT INTO aff VALUES ('100'), ('abc');
SELECT x, typeof(x) FROM aff;   -- 100|integer  and  abc|text
```

So "the column is INTEGER" never means "only integers get stored". Only `STRICT` gives you that.

## STRICT tables

`STRICT` (3.37+) enforces the declared type at write time. Allowed types: `INTEGER`, `REAL`, `TEXT`,
`BLOB`, `ANY`.

```sql
CREATE TABLE orders (
    id INTEGER PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id),
    total_cents INTEGER NOT NULL CHECK (total_cents >= 0),
    notes TEXT,
    created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))
) STRICT;
```

What STRICT does and does not do (all verified):

- **Does** reject non-lossless conversions: `INSERT ... (n) VALUES (1.5)` into an `INTEGER` column fails
  with `cannot store REAL value in INTEGER column`.
- **Does** implicitly force `NOT NULL` on primary key columns. A non-STRICT rowid table still accepts
  `NULL` in a `TEXT PRIMARY KEY`; STRICT does not.
- **Does not** forbid lossless numeric text: `'123'` inserted into an `INTEGER` column is stored as the
  integer `123`. Use `ANY` when you genuinely want mixed types, plus a `CHECK` for the shape you need.
- **Cannot** be added to an existing table; converting is a table-rebuild migration.

Use STRICT for **all new tables** (**SHOULD**) unless the target build is older than 3.37 or the table
must hold deliberately mixed types. For legacy databases, add STRICT only together with data cleaning, and
never mix STRICT and non-STRICT for the same logical entity without a comment explaining why.

## Storage conventions for scalars

### Dates and times

```sql
-- CORRECT: ISO-8601 text, UTC, with a fixed format
CREATE TABLE events (
    id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    event_date TEXT NOT NULL,     -- '2024-03-15'
    starts_at TEXT NOT NULL,      -- '2024-03-15T09:00:00.000Z'
    created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))
) STRICT;

SELECT title FROM events
WHERE event_date BETWEEN date('now') AND date('now', '+7 days');
```

ISO-8601 TEXT sorts lexicographically in time order, is readable in a debugger, and works with `date()`,
`datetime()`, `strftime()`, `unixepoch()`, `julianday()`. Store UTC and convert at the edges.

```sql
-- ACCEPTABLE: integer epoch seconds, when the project already does this or space is critical
-- Convert at the boundary: SELECT strftime('%Y-%m-%dT%H:%M:%fZ', raw, 'unixepoch');
-- WRONG: ambiguous local formats ('03/15/2024') - do not sort, do not parse, misread across locales
```

**Consistency beats preference**: if the schema already stores epoch integers, keep that for new columns.

### Booleans

```sql
CREATE TABLE features (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    is_enabled INTEGER NOT NULL DEFAULT 0 CHECK (is_enabled IN (0, 1))
) STRICT;
```

Never store `'true'`/`'yes'` in a TEXT column. `CHECK (x IN (0,1))` is what makes the intent enforceable.

### Money

```sql
CREATE TABLE line_items (
    id INTEGER PRIMARY KEY,
    product_id INTEGER NOT NULL REFERENCES products(id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price_cents INTEGER NOT NULL CHECK (unit_price_cents >= 0),
    total_cents INTEGER GENERATED ALWAYS AS (quantity * unit_price_cents) STORED
) STRICT;
```

Integers in minor units, always - this is a **MUST** for monetary values. REAL cannot represent decimal
fractions exactly - `SELECT 0.07*100;` returns `7.000000000000001` - and errors compound in sums. Format
at the edge with `printf('%.2f', total_cents/100.0)`.

### UUIDs

```sql
-- Canonical lowercase text UUID
CREATE TABLE resources (
    id TEXT PRIMARY KEY NOT NULL CHECK (
        length(id) = 36
        AND id GLOB '????????-????-????-????-????????????'
        AND lower(id) = id
        AND replace(id, '-', '') NOT GLOB '*[^0-9a-f]*'
    ),
    name TEXT NOT NULL
) STRICT;

-- Compact BLOB UUID (16 bytes)
CREATE TABLE blobs (
    id BLOB PRIMARY KEY NOT NULL CHECK (length(id) = 16),
    name TEXT NOT NULL
) STRICT;
```

`GLOB '????????-...'` alone only proves length and hyphen positions - it happily accepts
`3F2504E0-...` or non-hex characters. The `lower()` + `NOT GLOB '*[^0-9a-f]*'` pair is what makes it
hex (verified). Choose TEXT for debuggability, BLOB for storage; either way, be consistent per table.

### Closed value sets

Use `CHECK (col IN (...))` for small, stable sets; move to a lookup table when the set is edited by users,
needs ordering/labels, or is referenced by more than one table.

## Primary keys and rowids

```sql
-- CORRECT: default choice - rowid alias, auto-assigned, no extra index
CREATE TABLE logs (
    id INTEGER PRIMARY KEY,
    message TEXT NOT NULL,
    created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))
) STRICT;
```

`INTEGER PRIMARY KEY` *is* the rowid. Deleted rowids are reused when the table empties out (verified:
after `DELETE` of all rows the next insert got `id = 1`).

```sql
-- ACCEPTABLE: only when IDs must never be reused (external references, sync, audit)
CREATE TABLE audit (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    action TEXT NOT NULL
) STRICT;
```

`AUTOINCREMENT` guarantees monotonic, never-reused ids (verified: `id = 4` after deleting three rows). Its
cost is an extra `sqlite_sequence` row update per insert and a table you must not lose during backups.
That cost is small; the real argument is that most schemas do not need the guarantee. Decide by asking
"would reuse corrupt something outside this database?", not by habit.

```sql
-- CORRECT: WITHOUT ROWID for small rows keyed by text or by a composite key
CREATE TABLE settings (
    setting_key TEXT PRIMARY KEY NOT NULL,
    setting_value TEXT NOT NULL
) STRICT, WITHOUT ROWID;

CREATE TABLE tag_assignments (
    item_id INTEGER NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    tag_id INTEGER NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (item_id, tag_id)
) STRICT, WITHOUT ROWID;
```

In `WITHOUT ROWID` tables the primary key is implicitly `NOT NULL` (verified) and the table *is* the
index. Do not use it with a single `INTEGER PRIMARY KEY`: you lose the rowid fast paths and gain nothing.
Avoid it for wide rows with large payloads.

## Naming and structure

- Tables: `snake_case`, plural (`users`, `order_items`).
- Columns: `snake_case`; foreign keys end in `_id` (`user_id`).
- Indexes: `idx_<table>_<columns>`; unique indexes `uq_<table>_<columns>`.
- Triggers: `<table>_<event>` (`documents_ai`, `documents_ad`, `documents_au`).
- Join tables: `<a>_<b>` or a domain name; keep the pair columns identical to the referenced PK names.
- Avoid SQL keywords as identifiers (`order`, `group`, `index`, `where`). Quoting buys working SQL but
  leaks quotes into every query and every host-language binding. `key` and `value` are *not* reserved and
  are fine unquoted.
- Name columns after meaning, not type (`total_cents`, not `total_int`).

## Constraints and integrity

- `NOT NULL` by default; make nullability a documented decision, not an accident.
- `CHECK` for single-row domain rules: ranges, non-negative amounts, `IN (...)` sets, mutual exclusion.
  `CHECK` cannot reference other rows or tables.
- `UNIQUE` for natural keys; use a **partial unique index** to scope uniqueness (`WHERE deleted_at IS NULL`).
- Foreign keys: always `REFERENCES` with an explicit `ON DELETE`/`ON UPDATE` (`CASCADE` for owned
  children, `SET NULL` for optional links, `RESTRICT` to forbid orphaning). `DEFERRABLE INITIALLY
  DEFERRED` defers enforcement to `COMMIT` (verified) - use it for cyclic references, not as a shortcut.
- Index the child side of every foreign key; SQLite does not create that index for you.
- Audit once per session, not per query:

```sql
PRAGMA foreign_key_check;   -- returns (table, rowid, parent, fkid) for every violation; empty = clean
PRAGMA quick_check;
```

- Do not treat `CHECK` as input validation. `CHECK (email LIKE '%@%.%')` accepts `a@.c` and `a b@c.d`
  (verified). Validate in the application; keep the constraint as a backstop against buggy writers.

## SQL features worth using

### UPSERT (3.24+)

```sql
INSERT INTO settings (setting_key, setting_value)
VALUES ('theme', 'dark')
ON CONFLICT (setting_key) DO UPDATE SET setting_value = excluded.setting_value;
```

Idempotent writes without a read-modify-write race. `ON CONFLICT ... DO NOTHING` for "insert if absent".
The conflict target must be a real unique constraint or index.

### RETURNING (3.35+)

```sql
INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com')
RETURNING id, created_at;
```

Removes the follow-up `SELECT` (and its race) for generated ids/defaults. Works on `UPDATE`/`DELETE` too.

### Window functions (3.25+)

```sql
SELECT name, department, salary,
       RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rank,
       AVG(salary) OVER (PARTITION BY department) AS dept_avg
FROM employees;
```

Note `RANK()` leaves gaps on ties; `DENSE_RANK()`/`ROW_NUMBER()` behave differently. Pick intentionally.

### JSON (3.38+ built in; JSONB 3.45+)

```sql
CREATE TABLE events (
    id INTEGER PRIMARY KEY,
    payload TEXT NOT NULL CHECK (json_valid(payload)),
    event_type TEXT GENERATED ALWAYS AS (json_extract(payload, '$.type')) STORED,
    created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))
) STRICT;

CREATE INDEX idx_events_type ON events(event_type);
```

```sql
-- WRONG: filtering on a JSON expression - full scan plus per-row parse
SELECT * FROM events WHERE json_extract(payload, '$.type') = 'purchase';
```

Verified plans: the generated column uses `SEARCH events USING INDEX idx_events_type`, the raw
`json_extract` predicate gives `SCAN events`. If you must store JSONB blobs, declare the column `ANY`
in a STRICT table and use `CHECK (json_valid(doc, 6))`, because `json_valid()` on a BLOB returns `0`
(verified).

Use JSON for genuinely schemaless payloads, not to avoid modeling. Anything you filter or sort on belongs
in a real column (generated or not).

### FTS5 (3.9+)

External-content FTS5 avoids storing the text twice. It needs sync triggers - the virtual table alone
does **not** follow the content table.

```sql
CREATE TABLE documents (
    id INTEGER PRIMARY KEY,
    title TEXT NOT NULL,
    body TEXT NOT NULL
) STRICT;

CREATE VIRTUAL TABLE documents_fts USING fts5(
    title, body,
    content='documents',
    content_rowid='id',
    tokenize='porter unicode61'
);

CREATE TRIGGER documents_ai AFTER INSERT ON documents BEGIN
    INSERT INTO documents_fts(rowid, title, body) VALUES (new.id, new.title, new.body);
END;

CREATE TRIGGER documents_ad AFTER DELETE ON documents BEGIN
    INSERT INTO documents_fts(documents_fts, rowid, title, body)
    VALUES ('delete', old.id, old.title, old.body);
END;

CREATE TRIGGER documents_au AFTER UPDATE ON documents BEGIN
    INSERT INTO documents_fts(documents_fts, rowid, title, body)
    VALUES ('delete', old.id, old.title, old.body);
    INSERT INTO documents_fts(rowid, title, body) VALUES (new.id, new.title, new.body);
END;

-- Maintenance commands (all verified)
INSERT INTO documents_fts(documents_fts) VALUES ('rebuild');          -- after a bulk load
INSERT INTO documents_fts(documents_fts) VALUES ('integrity-check');
INSERT INTO documents_fts(documents_fts) VALUES ('optimize');

SELECT title, bm25(documents_fts) AS score
FROM documents_fts WHERE documents_fts MATCH 'borrow' ORDER BY score;
```

```sql
-- WRONG: substring search over the content table
SELECT * FROM documents WHERE body LIKE '%search term%';
```

Triggers are the failure mode to watch: add a schema test that inserts, updates, and deletes a row and
asserts the FTS row count matches. Skipping the triggers silently desynchronizes the index.

### Generated columns (3.31+)

`STORED` costs space and write time but can be indexed and read cheaply; `VIRTUAL` costs nothing to store
but is recomputed per row. Note that `ALTER TABLE ... ADD COLUMN` can add a `VIRTUAL` generated column
but cannot add a `STORED` one to a non-empty table (verified: `cannot add a STORED column`).

## Transactions and concurrency

```sql
-- CORRECT: take the write lock up front
BEGIN IMMEDIATE;
-- ... read, decide, write ...
COMMIT;
```

```sql
-- WRONG: deferred write transaction
BEGIN;                      -- read snapshot only
SELECT ...;                 -- another writer commits here
INSERT INTO ...;            -- may now fail with SQLITE_BUSY / "database is locked"
COMMIT;
```

In WAL a deferred transaction that starts as a read and then writes must upgrade its snapshot; if another
writer committed in between, the upgrade fails (verified: `database is locked` despite a busy timeout).
`BEGIN IMMEDIATE` avoids the failure by queueing writers at `BEGIN`.

Rules:

- **Writes (MUST)**: `BEGIN IMMEDIATE` + `busy_timeout` (5000ms) + a retry loop on `SQLITE_BUSY`.
- **Reads**: plain `BEGIN` gives a consistent snapshot across several statements; keep it short so
  checkpointing is not blocked.
- **Connections**: the safe default is one writer connection plus N reader connections per process,
  all configured with the same PRAGMAs. WAL allows more writers, but they serialize; do not build a pool
  that pretends otherwise.
- **Cleanup**: close every connection on shutdown. A leaked connection holds locks and blocks
  checkpointing; use context managers / RAII wrappers, not manual closes in happy paths only.
- **Busy handling**: `SQLITE_BUSY_SNAPSHOT` (WAL write upgrade) needs a full transaction rollback and
  retry; retrying the statement alone will not help.

## Indexing

- Index the columns used by `WHERE`, `JOIN`, and `ORDER BY` on hot paths - and the child side of every
  foreign key.
- Composite indexes serve a leftmost prefix only: `(a, b)` covers `a` and `a, b`, not `b` alone.
- Partial indexes for filtered access: `CREATE INDEX idx_open ON tasks(due_at) WHERE done_at IS NULL;`
- Expression indexes for computed predicates: `CREATE INDEX idx_lower_email ON users(lower(email));`
- Covering index (all selected columns present) removes the table lookup; a narrow table may not need it.
- Every index taxes writes and storage. Add one because a plan says so, not by default.
- Verify, do not guess:

```sql
EXPLAIN QUERY PLAN SELECT ...;   -- want SEARCH ... USING INDEX / COVERING INDEX, not SCAN
ANALYZE;                          -- or PRAGMA optimize on connection close
```

## Maintenance, backups, migrations

- **Schema version**: keep an integer in `PRAGMA user_version` (verified writable) or in a
  `schema_migrations` table, and fail fast on mismatch.
- **File identity**: `PRAGMA application_id` tags the file type so tools do not open the wrong file.
- **Backup**: `VACUUM INTO 'backup.db'` produces a consistent, compact copy while the database is live
  (3.27+). Do not copy a live database file with the filesystem - WAL contents are not included.
- **Integrity**: `PRAGMA quick_check` on startup for critical data, `PRAGMA integrity_check` when
  diagnosing corruption.
- **Migrations**: `ALTER TABLE` is deliberately limited (rename table/column, add column, drop column).
  Anything else is the documented 12-step rebuild: `PRAGMA foreign_keys = OFF` **outside** the transaction,
  `BEGIN`, create the new table, copy the rows, drop the old table, rename, recreate indexes/triggers/views,
  `PRAGMA foreign_key_check`, `COMMIT`, then `PRAGMA foreign_keys = ON` again.
- **Adding a `NOT NULL` column** to an existing table requires a default, or the migration must backfill
  before enforcing.

## Verification checklist

- [ ] `sqlite_version()` and `pragma_compile_options` recorded before choosing features
- [ ] Every connection applies the per-connection PRAGMAs; `PRAGMA foreign_keys` returns `1`
- [ ] `journal_mode` is `wal` on a local filesystem, or `delete` deliberately
- [ ] `synchronous` chosen for the durability the data needs
- [ ] New tables are `STRICT` (or the reason not to be is written down)
- [ ] Dates, booleans, money, and UUIDs use one documented representation per database
- [ ] Primary keys chosen deliberately; `AUTOINCREMENT` justified where used
- [ ] `NOT NULL`, `CHECK`, `UNIQUE`, and FK actions cover the invariants
- [ ] FK child columns indexed; `PRAGMA foreign_key_check` clean
- [ ] JSON filters served by generated columns, not `json_extract` in `WHERE`
- [ ] FTS5 tables have insert/update/delete triggers and a test that proves they stay in sync
- [ ] Write transactions use `BEGIN IMMEDIATE`; `SQLITE_BUSY` handling exists
- [ ] Plans checked with `EXPLAIN QUERY PLAN`; `ANALYZE`/`PRAGMA optimize` scheduled
- [ ] Backup path uses `VACUUM INTO`, not a filesystem copy of a live file

## Compatibility matrix

Minimum version for each feature; confirm against the Step 0 probe before relying on it.

| Feature | Minimum version |
| --- | --- |
| WAL mode | 3.7.0 |
| FTS5 | 3.9.0 |
| `PRAGMA optimize` | 3.18.0 |
| UPSERT | 3.24.0 |
| Window functions | 3.25.0 |
| `VACUUM INTO` | 3.27.0 |
| Generated columns | 3.31.0 |
| `PRAGMA analysis_limit` | 3.32.0 |
| RETURNING clause | 3.35.0 |
| Math functions (`SQLITE_ENABLE_MATH_FUNCTIONS`) | 3.35.0 |
| STRICT tables | 3.37.0 |
| Built-in JSON, `->` / `->>` operators, `unixepoch()` | 3.38.0 |
| RIGHT / FULL OUTER JOIN | 3.39.0 |
| JSONB | 3.45.0 |

Mobile platforms and embedded systems often ship older builds; the version probe is the only reliable
answer.

## References

- `references/coding_style.md` - the SQLite/Zipvfs coding style these conventions are distilled from
  ([source](https://sqlite.org/zipvfs/doc/trunk/www/style.wiki), retrieved 2026-11-17). It governs C
  source, so only its transferable rules are applied here:

  | Reference rule (C) | SQLite equivalent used in this skill |
  | --- | --- |
  | Document purpose, preconditions, and invariants at the definition | Header comment on each table; `NOT NULL`/`CHECK` that encode the invariant |
  | Verify assumptions with `assert()` | `CHECK` constraints, `STRICT`, `PRAGMA foreign_key_check`, schema tests |
  | All code subblocks enclosed in `{...}`; no implicit control flow | Explicit transactions (`BEGIN IMMEDIATE`/`COMMIT`) instead of autocommit side effects |
  | No resource leaks under error conditions | Close connections on all paths; roll back and retry on `SQLITE_BUSY` |

- `references/verify.sql` - runnable probe of every version gate, PRAGMA default, and claim marked
  "verified" above.
- SQLite documentation: `sqlite.org/pragma.html`, `sqlite.org/datatype3.html`,
  `sqlite.org/stricttables.html`, `sqlite.org/lang_transaction.html`, `sqlite.org/fts5.html`,
  `sqlite.org/json1.html`, `sqlite.org/wal.html`, `sqlite.org/lang_altertable.html`.

## Boundaries

- Normalization, relationship modeling, migration strategy at the design level → `database-design`
- PostgreSQL types, functions, `EXPLAIN`, VACUUM, index tuning → `postgresql-patterns`,
  `postgresql-optimization`
- Data access from application code (pools, ORMs, query builders) → `python-backend-design`,
  `rust-backend-design`
- Input validation, secrets, injection review → `security-review`
