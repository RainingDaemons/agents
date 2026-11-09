---
name: rust-backend-design
description: Architecture and data-access patterns for this backend — request flow, handler-owned DB access, bb8/tokio-postgres vs sqlx pools, query and transaction patterns, Apalis background jobs, config/secrets, and tracing. Use when adding handlers, touching the DB from routers, wiring the job queue, or reviewing backend structure.
license: MIT
metadata:
  author: rainingdaemons/agents
  version: "1.0"
---

# Backend Patterns (Axum)

How this backend is structured and how to add to it without fighting its design. It describes the **architecture and data access**; axum wiring is in `axum-patterns`, REST contract design in `api-design`, SQL schema/index review in `postgresql-patterns` / `postgresql-optimization` / `database-design`, Rust language mechanics in `rust-patterns`.

## When to activate

- Adding or changing a handler that talks to the database
- Deciding where business logic lives
- Writing queries, avoiding N+1, or using a transaction
- Enqueuing or processing a background job
- Reading configuration / secrets

## Request flow (no service or repository layer)

```
HTTP → Router → AuthLayer (decode JWT → AuthUser extension) → RequireRoleLayer (RBAC check) → Handler
```

- **Handlers own their DB access directly.** There is deliberately **no** separate service or repository layer (see `agentwork/docs/architecture.md` §3). Keep business logic in the handler; do not introduce repository traits, `ProductService`, or a cache decorator.
- One file per resource in `src/routers/`. DB access lives there and in `connection.rs` — nowhere else.
- Auth, RBAC, and crypto live in `src/common/` (`middleware.rs`, `crypto.rs`).

## Two connection pools (do not mix them)

Typed in `src/connection.rs`:

```rust
pub type ConnectionPool = bb8::Pool<PostgresConnectionManager<NoTls>>; // handlers

pub async fn create_pool() -> ConnectionPool;        // bb8 + tokio-postgres
pub async fn create_sqlx_pool() -> sqlx::PgPool;     // Apalis queue ONLY
```

- **Handlers** use `ConnectionPool` (bb8 + `tokio-postgres`, runtime queries, no ORM).
- **`sqlx::PgPool`** is used **only** by the Apalis job queue storage. Do not use it for request-path queries.
- The pool is passed as `State` and is internally reference-counted — no `Arc` wrapper needed.

```rust
async fn get_all_products(
    State(pool): State<ConnectionPool>,
) -> Result<Json<ProductsListResponse>, (StatusCode, String)> {
    let conn = pool.get().await.map_err(internal_error)?;
    // ...
}
```

## Query patterns

Run queries on a pooled connection with `query` / `query_opt` / `query_one` / `execute`, mapping errors through the resource's `internal_error` helper.

- **Select only the columns you need** — avoid `SELECT *` (wide rows, TOAST fetches, serialization cost).
- **Avoid N+1:** fetch related rows in one JOIN, or batch with `WHERE id = ANY($1)` and build a `HashMap` in memory. See `products.rs::get_all_products` for a batched companion fetch.
- **Parameterize** with `$1, $2, …`; never format values into the SQL string. Own the parameter typing explicitly.
- **Soft deletes are the norm:** filter `is_deleted = false`.

```rust
let rows = conn
    .query(
        "SELECT id, name, slug FROM categories WHERE is_deleted = false ORDER BY sort_order, name",
        &[],
    )
    .await
    .map_err(internal_error)?;
```

## Transactions

`tokio-postgres` transactions borrow a connection and roll back on drop unless committed:

```rust
let mut conn = pool.get().await.map_err(internal_error)?;
let tx = conn.transaction().await.map_err(internal_error)?;
tx.execute("INSERT INTO orders (user_id, total) VALUES ($1, $2)", &[&user_id, &total])
    .await.map_err(internal_error)?;
tx.commit().await.map_err(internal_error)?;
```

Multi-statement writes that must be atomic should use a transaction. Several existing handlers issue sequential `execute` calls without one — when you touch such a handler, wrap the group in a transaction.

## Background jobs (Apalis + Postgres)

Email is never sent inline from a handler. Enqueue it; an Apalis worker delivers it.

- Storage: `apalis_postgres::PostgresStorage`; worker wiring in `src/queue/worker.rs`.
- Delivery is behind the `EmailSender` trait (`NoopEmailSender` in dev) — handlers stay testable.
- Worker layers: `JobTracingLayer`, `TimeoutLayer`, `ConcurrencyLimitLayer`; graceful shutdown via `tokio::select!` on `ctrl_c`.
- Enqueue via `storage.push(job)`. Job types live in `queue/types.rs`; handlers in `queue/handlers.rs`.

`tokio::spawn` is acceptable only for fire-and-forget work where losing the job on crash is fine; durable work goes through the queue.

## Config, secrets & PII

- Read configuration from the environment (`DB_URL`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`, `JWT_SECRET`, `RUT_ENCRYPTION_KEY`); `dotenvy::dotenv()` loads `backend/.env`.
- Never hardcode secrets or default them to real values. `main.rs` bootstrapping may `expect()` on missing critical vars; handlers may not.
- **Never store PII (RUT, etc.) in plaintext** — use `common/crypto.rs` (AES-256-GCM).

## Logging & observability

- Use the `tracing` crate with structured fields; `println!` / `dbg!` are banned in production code (`connection.rs` bootstrap prints are a known exception to clean up).
- Log the real error server-side; return a generic message to the client.

## Error handling

Handlers return `Result<Success, Rejection>`. Return a typed rejection, never `unwrap()`/`expect()` (except `main.rs` bootstrapping). Both rejection shapes and the target (log + generic message) are documented in `axum-patterns`.

## Module layout

```
src/main.rs        tracing init, env, pools, worker spawn, router assembly, bind :3000
src/lib.rs         library root for tests
src/connection.rs  create_pool() + create_sqlx_pool()
src/common/        crypto.rs, middleware.rs (JwtClaims, AuthUser, Role, layers)
src/queue/         Apalis email queue (worker, handlers, types, layers, sender trait)
src/routers/       one module per resource
```

## Checklist

- [ ] DB access stays in `src/routers/` + `connection.rs`; no new service/repository layer
- [ ] Handlers use `ConnectionPool`; `sqlx` only for the queue
- [ ] Queries select explicit columns; no N+1; values parameterized
- [ ] Multi-statement writes that must be atomic use a transaction
- [ ] Email is enqueued, not sent inline
- [ ] Secrets/PII come from env / `common/crypto.rs`; none hardcoded
- [ ] `tracing` used; no `println!`/`dbg!`; no `unwrap()`/`expect()` in handlers
