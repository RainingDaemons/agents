---
name: postgresql-patterns
description: PostgreSQL engine and language features - advanced data types (JSONB, arrays, ranges, geometric, custom types/domains/ENUM), extensions, window functions, CTEs, full-text search, PL/pgSQL, and error reporting (RAISE / SQLSTATE). Use when writing PostgreSQL DDL/DML that should use engine-specific capabilities or writing PL/pgSQL error messages.
license: MIT
metadata:
  author: rainingdaemons/agents
  version: "1.0"
  source: Generated from PostgreSQL v16, following PostgreSQL Coding Conventions and the PostgreSQL Style Guide.
---

# PostgreSQL Patterns

PostgreSQL-the-engine guidance: the data types, functions, extensions, and SQL/PL-pgSQL features that are unique to PostgreSQL. This skill owns **what PostgreSQL can do**. Query planning and indexing strategy live in `postgresql-optimization`; schema/normalization/constraints in `database-design`.

> **This repo overrides the generic examples below.** Follow `internal/conventions/DATABASE_CONVENTIONS.md`: `UUID PRIMARY KEY DEFAULT gen_random_uuid()`, `TIMESTAMPTZ`, soft deletes via `is_deleted`, money as `BIGINT`. The examples here use those types already.

## When to activate

- Choosing an engine-specific data type (JSONB, arrays, ranges, geometric, custom types)
- Using window functions, CTEs, or full-text search
- Enabling or calling an extension
- Writing PL/pgSQL functions or triggers
- Writing PL/pgSQL error messages (`RAISE`, SQLSTATE codes)

## Formatting

- Indent SQL and PL/pgSQL with 4 spaces per level (like the tables in this skill); never mix tabs and spaces in one file.
- Keep lines readable within ~80 columns; break long `SELECT` lists and `CREATE TABLE` column definitions one item per line.
- Follow BSD-style brace/`BEGIN … END` placement: each block's `BEGIN`/`END` on its own line.
- The strongest rule is local consistency: **make new SQL look like the SQL around it**. Align columns only if the file already does.
- Never comment with `//`; use `--` for line comments and `/* … */` for blocks.

## JSONB

Prefer JSONB over JSON (it is stored decomposed and indexable). Query with containment/path operators, not text casting.

```sql
CREATE TABLE orders (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    data jsonb NOT NULL DEFAULT '{}',
    created_at timestamptz NOT NULL DEFAULT now()
);

-- Indexable containment query
SELECT * FROM orders WHERE data @> '{"status": "shipped"}';

-- Path extraction
SELECT data #>> '{user,role}' FROM orders;

-- Containment + path filter
SELECT * FROM orders
WHERE data @> '{"type": "login"}'
  AND data #>> '{user,role}' = 'admin';

-- Aggregate JSON
SELECT jsonb_agg(data) FROM orders WHERE data ? 'user_id';
```

- Keep JSONB structured and validated with `CHECK` constraints where the shape is fixed.
- For query performance of JSONB (GIN index, `@>` vs `->>`), see `postgresql-optimization`.

## Arrays

```sql
CREATE TABLE posts (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tags text[],
    category_ids integer[]
);

SELECT * FROM posts WHERE 'postgresql' = ANY(tags);            -- membership
SELECT * FROM posts WHERE tags && ARRAY['database', 'sql'];    -- overlap
SELECT * FROM posts WHERE array_length(tags, 1) > 3;
SELECT array_agg(DISTINCT c) FROM posts, unnest(category_ids) AS c;
```

Prefer a normalized join table when you need referential integrity or frequent joins on the elements; arrays suit small, unordered, read-mostly sets.

## Custom types, domains & enums

```sql
CREATE TYPE order_status AS ENUM ('pending', 'processing', 'shipped', 'delivered', 'cancelled');

CREATE TYPE address_type AS (
    street text,
    city text,
    postal_code text,
    country text
);

CREATE DOMAIN email_address AS text
    CHECK (VALUE ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');

CREATE TABLE customers (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    email email_address NOT NULL,
    address address_type,
    status order_status DEFAULT 'pending'
);
```

- Define enums **before** the tables that reference them.
- Domains package a reusable `CHECK`; use them instead of repeating the same constraint.
- Enum values are versioned with migrations — adding values is a minor change; removing/renaming is breaking.

## Range types

```sql
CREATE TABLE reservations (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    room_id integer,
    reservation_period tstzrange,
    price_range numrange
);

-- Overlap query
SELECT * FROM reservations
WHERE reservation_period && tstzrange('2024-07-20', '2024-07-25');

-- Prevent overlapping reservations (needs GiST; see postgresql-optimization)
ALTER TABLE reservations
ADD CONSTRAINT no_overlap
EXCLUDE USING gist (room_id WITH =, reservation_period WITH &&);
```

## Geometric types

```sql
CREATE TABLE locations (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    name text,
    coordinates point,
    coverage circle,
    service_area polygon
);

SELECT name FROM locations
WHERE coordinates <-> point(40.7128, -74.0060) < 10;   -- distance
```

## Window functions & analytics

```sql
SELECT
    product_id,
    sale_date,
    amount,
    SUM(amount)  OVER (PARTITION BY product_id ORDER BY sale_date)                                  AS running_total,
    AVG(amount)  OVER (PARTITION BY product_id ORDER BY sale_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg,
    DENSE_RANK() OVER (PARTITION BY EXTRACT(month FROM sale_date) ORDER BY amount DESC)             AS monthly_rank,
    LAG(amount, 1) OVER (PARTITION BY product_id ORDER BY sale_date)                                AS prev_amount
FROM sales;
```

## CTEs (including recursive)

```sql
-- Recursive query for hierarchical data
WITH RECURSIVE category_tree AS (
    SELECT id, name, parent_id, 1 AS level
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    SELECT c.id, c.name, c.parent_id, ct.level + 1
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY level, name;
```

CTEs are for readability and recursion. Whether a CTE is inlined or materialized (and the performance implications) is covered in `postgresql-optimization`.

## Full-text search

```sql
CREATE TABLE documents (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    title text,
    content text,
    search_vector tsvector
);

UPDATE documents
SET search_vector = to_tsvector('english', title || ' ' || content);

SELECT *
FROM documents
WHERE search_vector @@ plainto_tsquery('english', 'postgresql database');

-- Ranked results
SELECT *, ts_rank(search_vector, plainto_tsquery('postgresql')) AS rank
FROM documents
WHERE search_vector @@ plainto_tsquery('postgresql')
ORDER BY rank DESC;
```

A `GIN` index on `search_vector` is required for speed — see `postgresql-optimization`.

## Extensions

```sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";    -- gen_random_uuid(), crypt(), gen_salt()
CREATE EXTENSION IF NOT EXISTS "unaccent";    -- accent-insensitive text
CREATE EXTENSION IF NOT EXISTS "pg_trgm";     -- trigram similarity / LIKE indexes

SELECT gen_random_uuid();
SELECT crypt('password', gen_salt('bf'));     -- password hashing
SELECT similarity('postgresql', 'postgersql');
```

Use `IF NOT EXISTS`, and only extensions the deployment actually provides.

## PL/pgSQL functions & triggers

```sql
-- Keep the trigger body minimal; let the WHEN clause gate it.
CREATE OR REPLACE FUNCTION update_modified_time()
RETURNS trigger AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_modified_time_trigger
    BEFORE UPDATE ON products
    FOR EACH ROW
    WHEN (OLD.* IS DISTINCT FROM NEW.*)   -- fire only on real changes
    EXECUTE FUNCTION update_modified_time();
```

- Prefer set-based SQL over row-by-row loops in functions.
- Use `RETURNS trigger` functions for audit/`updated_at` maintenance.

## Error reporting (`RAISE`) and message style

`RAISE` is the SQL-level equivalent of the server's `ereport`: same three-part message model, same style guide. Pick a real severity (`DEBUG`, `LOG`, `INFO`, `NOTICE`, `WARNING`, `EXCEPTION`); `EXCEPTION` aborts the statement, the others return normally.

```sql
-- User-facing: short lowercase primary message + DETAIL (facts) + HINT (fix).
RAISE EXCEPTION 'order total cannot be negative'
    USING ERRCODE = '23514',
          DETAIL  = format('Order %s has line totals summing to %s.', p_order_id, v_total),
          HINT    = 'Correct the order items and retry.';

-- Retryable failure: past tense, include the underlying reason.
RAISE WARNING 'could not lock order row %: %', p_order_id, SQLERRM
    USING HINT = 'Retry the transaction.';

-- Impossible branch: internal message, no translation/hint needed.
RAISE EXCEPTION 'unrecognized order status: %', v_status
    USING ERRCODE = '22023';
```

Message rules (from PostgreSQL's error message style guide):

- **Primary message**: short, factual, lowercase first letter, no trailing period, never `!`. No implementation details or function names — say what the code was trying to do.
- **DETAIL**: the facts (offending value, object kind). **HINT**: what to do to fix it, even if it does not always apply. Both are complete sentences, capitalized, each ending with a period.
- **Always state the reason.** `could not open file x` is wrong; `could not open file x: permission denied` is right.
- **Past vs present tense**: `could not …` when retrying might succeed; `cannot …` when it is permanently impossible.
- **Active voice**, no `I`; the program is not human. Avoid contractions (`cannot`, not `can't`).
- **Name the object type**: "relation %s", "column %s", "data type %s" — not bare `foo.bar`.
- **Quote embedded names and user-supplied values** that may contain words; do not quote operators or the output of helpers that already quote.
- **Set `ERRCODE`** (SQLSTATE) on every exception you raise; it is the machine-readable contract clients rely on (see `api/` error mapping).
- **Word choice**: replace `unable` with `cannot`/`could not`, `bad`/`illegal` with the actual defect, `unknown` with `unrecognized` (and include the value), and `non-negative` with `greater than or equal to zero`. Spell words out (specification, authentication, statistics, parentheses).
- **Formatting-neutral text**: no tabs, no trailing newline; clients and the log wrap lines. A `\n` may separate paragraphs only.
- In `RAISE`, `%` consumes the next argument and `%%` is a literal percent sign (`%s` is not a `RAISE` placeholder — use `format()` in `DETAIL`/`HINT` if you need typed formatting). `SQLERRM` and `SQLSTATE` expose the caught error's message and code.

## Documenting SQL

When documenting a function, query, or migration (in comments or a spec), follow PostgreSQL's own reference-page layout — a fixed section order so readers always find the parts.

Order: **Synopsis → Description → Parameters → Outputs → Notes → Examples → See Also** (add **Compatibility** for standards/portability notes; **History** only if there are milestones).

```sql
-- Function: order_total(order_id)
--
-- Synopsis:  order_total(p_order_id uuid) RETURNS bigint
-- Returns:   sum of line totals in the smallest currency unit (CLP); NULL if the order has no items.
--
-- Notes:     Money is stored as bigint (see DATABASE_CONVENTIONS); no rounding here.
--            Uses the order_items(order_id) index.
--
-- Example:   SELECT order_total('00000000-0000-0000-0000-000000000000');
CREATE FUNCTION order_total(p_order_id uuid) RETURNS bigint AS $$
    SELECT sum(unit_price * quantity) FROM order_items WHERE order_id = p_order_id;
$$ LANGUAGE sql STABLE;
```

- Keep the *what* in the code; put the *why* (caveats, invariants, security, performance) in **Notes**.
- One runnable **Example** beats a paragraph.

## Checklist

- [ ] JSONB used (not JSON) and queried with `@>` / `#>>`, not `::text LIKE`
- [ ] Arrays only for small read-mostly sets; join tables for relational data
- [ ] Enums defined before use; constraints/domains replace repeated `CHECK`s
- [ ] Range/geometric types used where they fit; exclusion constraints for overlaps
- [ ] Window functions / CTEs used instead of hand-rolled procedural loops
- [ ] FTS via `tsvector` + matched query functions
- [ ] Extensions created with `IF NOT EXISTS` and actually available
- [ ] PL/pgSQL kept set-based; trigger `WHEN` clauses avoid needless fires
- [ ] SQL/PL-pgSQL indented 4 spaces, ~80 columns, matching the surrounding file; no `//` comments
- [ ] `RAISE` messages: lowercase short primary text, complete-sentence DETAIL/HINT, stated reason, `ERRCODE` set
- [ ] Error wording follows the style guide (active voice, correct tense, no "unable"/"bad"/"illegal"/contractions)
- [ ] Functions/migrations documented in Synopsis → Description → Parameters → Outputs → Notes → Examples order
- [ ] Repo rules from `internal/conventions/DATABASE_CONVENTIONS.md` respected (UUID PKs, `TIMESTAMPTZ`, `is_deleted`, money as `BIGINT`)

## Boundaries

- **Index strategy, `EXPLAIN`, planner, pagination, VACUUM, `pg_stat_statements`** → `postgresql-optimization`
- **Schema/normalization, keys, constraints, migrations, RLS/security, retention** → `database-design`
- **Repo-specific schema rules** → `internal/conventions/DATABASE_CONVENTIONS.md`

## References

- `references/postgresql16_coding_conventions.md` — *PostgreSQL 16, Chapter 56: Coding Conventions* (formatting, `ereport`/`elog` severity and auxiliary fields, error message style guide, C/PL conventions). Source of the formatting and `RAISE` message rules above.
