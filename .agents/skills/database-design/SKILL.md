---
name: database-design
description: Relational database design patterns - normalization, keys, relationships, constraints, naming, timestamps, soft deletes, migration strategy, and security/retention. Use when designing or reviewing a schema.
license: MIT
metadata:
  author: rainingdaemons/agents
  version: "1.0"
---

# Database Design Patterns

How to structure a relational schema that stays correct as it grows. This skill owns **schema shape**: normalization, keys, relationships, constraints, migrations, and security design. It is engine-agnostic where it can be; PostgreSQL specifics belong to `postgresql-patterns` (types/functions) and `postgresql-optimization` (indexes/plans).

> **Repo rules override.** `internal/conventions/DATABASE_CONVENTIONS.md` is authoritative for this repository (UUID PKs, `TIMESTAMPTZ`, `is_deleted`, money as `BIGINT`, migration versioning). This skill gives the reasoning; that document gives the rule.

## When to activate

- Designing a new table or relationship
- Choosing a primary key, foreign key, or uniqueness constraint
- Deciding whether to normalize or deliberately denormalize
- Planning a migration (add/drop/rename, backfill)
- Reviewing schema for security, retention, or access patterns

## Normalization

- Aim for **third normal form** by default: every non-key column depends on the key, the whole key, and nothing but the key.
- Denormalize **only** for a measured read pattern, and document the reason + how the duplicate stays in sync (trigger, application, or materialized view).
- Separate concerns: don't mix an entity's attributes with a history/log of events; use a dedicated append-only table.

## Keys and relationships

- Prefer a **surrogate primary key** (stable, opaque) plus a `UNIQUE` **natural key** where one exists. In this repo the surrogate is `uuid` (see the conventions doc).
- UUID vs integer surrogate: UUIDs don't leak counts and are safe to generate client-side; integers are smaller/faster. This repo standardizes on UUID.
- **1:N** — the many side carries the FK. **M:N** — use a join table with a composite unique key on the pair.
- A **1:1** is a normal table with the FK also `UNIQUE`, or the columns extended onto the same table.
- Never use UUID non-enumerability as the *only* authorization control (see "Security").

```sql
-- M:N join table
CREATE TABLE product_categories (
    product_id  uuid NOT NULL REFERENCES products(id)  ON DELETE CASCADE,
    category_id uuid NOT NULL REFERENCES categories(id) ON DELETE CASCADE,
    PRIMARY KEY (product_id, category_id)
);
```

## Constraints

- Declare `NOT NULL` deliberately; most columns should be non-null.
- `UNIQUE` for natural keys; use a **partial unique index** to enforce uniqueness only over live rows (`WHERE is_deleted = false`).
- `CHECK` for value/range/cross-column rules the application should not be trusted to enforce alone.
- Define FK `ON DELETE`/`ON UPDATE` behavior explicitly: `CASCADE` for owned children, `SET NULL` for optional references, `RESTRICT` to forbid orphaning.
- Use an `EXCLUDE` constraint for "no overlap" invariants (e.g. reservations).

## Naming

- Tables `snake_case`, plural; columns `snake_case`; FKs named `<referenced>_id`.
- Be consistent across the schema — names are an API. (The repo pins exact prefixes for indexes/constraints in `DATABASE_CONVENTIONS.md`.)

## Timestamps, soft deletes, money

- Every table gets `created_at`; mutable tables get `updated_at` (both timezone-aware).
- Soft deletes: an `is_deleted` flag preserves history and satisfies referential integrity; pair with partial unique indexes.
- **Money:** store integer minor units (e.g. `BIGINT`), never floating point. `NUMERIC` is acceptable where the DB must do decimal math; the repo mandates `BIGINT`.

## Enums, domains, JSON

- Use a constrained type (enum/domain/`CHECK`) for closed value sets instead of a free-form string.
- Use JSON only for genuinely schemaless, document-shaped data — not as a shortcut to avoid modeling. Keep the shape validated and define which keys are queried.

## Indexes (design-level)

- Index every foreign key (the planner needs it for joins/deletes).
- Index columns used in hot `WHERE`/`ORDER BY`/`JOIN`. Whether to use composite/partial/covering indexes, and index-type choice, is `postgresql-optimization`'s job.

## Migrations

- **Never modify an applied migration**; add a new, versioned, incremental one.
- Wrap in a transaction; make it idempotent-safe (`IF NOT EXISTS` / `IF EXISTS`).
- **Backward-compatible by default:** add columns nullable or with defaults; never add a `NOT NULL` column without a default to an existing table. Use expand → backfill → contract for breaking changes.
- Record applied migrations to prevent re-application; run under a dedicated DDL role.
- Backfill large tables in batches to avoid long locks.

## Security, access, retention

- **Least privilege:** separate roles for app (no DDL), migrations (DDL), admin.
- **RLS** for row-level tenancy/multi-user isolation where the DB must enforce it.
- **Encryption at rest** for sensitive columns (application-level or `pgcrypto`); never store raw tokens or passwords — store salted hashes / token hashes.
- **Audit** admin/billing operations append-only; never update/delete audit rows.
- **Retention:** define what is kept and for how long; purge/anonymize on schedule and on data-subject request.
- Every read/write of customer-scoped data carries an ownership filter (`WHERE user_id = $authenticated_user`).

## Checklist

- [ ] Table is in 3NF, or denormalization is deliberate and documented
- [ ] Primary key chosen; natural key enforced with `UNIQUE` where applicable
- [ ] Relationships modeled (FK on the many side; join table for M:N)
- [ ] `ON DELETE`/`ON UPDATE` defined explicitly on every FK
- [ ] `NOT NULL`, `CHECK`, and uniqueness constraints cover the invariants
- [ ] `created_at` (and `updated_at`) present; timestamps timezone-aware
- [ ] Soft-delete and partial-unique strategy consistent
- [ ] Money stored as integer minor units
- [ ] FKs and hot columns indexed (tuning deferred to `postgresql-optimization`)
- [ ] Migration is new, atomic, idempotent, and backward-compatible
- [ ] Roles least-privilege; sensitive data encrypted; audit append-only; retention defined

## Boundaries

- **PostgreSQL types/functions (JSONB, arrays, ranges, FTS, window functions, extensions, PL/pgSQL)** → `postgresql-patterns`
- **Query performance, indexes, `EXPLAIN`, VACUUM** → `postgresql-optimization`
- **This repository's exact schema rules** → `internal/conventions/DATABASE_CONVENTIONS.md`
