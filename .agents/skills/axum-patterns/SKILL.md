---
name: axum-patterns
description: axum 0.8 framework patterns for this backend — routers, state, extractors, responses, middleware, and tower integration. Use when building or reviewing axum routes, handlers, extractors, `IntoResponse` impls, or middleware. Framework-focused; HTTP contract design lives in api-design and DB access in rust-backend-design.
license: MIT
metadata:
  author: rainingdaemons/agents
  version: "1.0"
---

# axum Patterns (axum 0.8)

Framework-level patterns for this backend. This skill owns **how to wire axum**: routers, state, extractors, responses, and middleware. It does not cover REST contract design (`api-design`), database access (`rust-backend-design`), or Rust language mechanics (`rust-patterns`).

## When to activate

- Defining or restructuring routers and route handlers
- Writing custom extractors or `IntoResponse` implementations
- Adding middleware with tower `Layer` or `axum::middleware::from_fn`
- Handling multipart uploads (`axum-extra`) or typed path/query params

## Router structure

Routers are typed by the state they carry. Split public vs. protected routes and merge them:

```rust
use axum::{Router, routing::{delete, get, patch, post}, extract::{Json, Path, Query, State}, http::StatusCode};
use crate::common::middleware::{AuthLayer, RequireRoleLayer, Role};
use crate::connection::ConnectionPool;

pub fn products_router(jwt_secret: Arc<str>) -> Router<ConnectionPool> {
    let public = Router::new()
        .route("/", get(get_all_products))
        .route("/{id}", get(get_product_by_id));

    // `.layer` applies to every route declared before it; order is bottom-up.
    let protected = Router::new()
        .route("/", post(create_product))
        .route("/{id}", delete(delete_product))
        .layer(RequireRoleLayer::new(Role::Admin))
        .layer(AuthLayer::new(jwt_secret));

    public.merge(protected)
}
```

- Path params use axum 0.8 syntax `/{id}` (not `/:id`).
- `nest("/prefix", router)` mounts a sub-router; `merge` combines two routers at the same base.
- Attach a `Layer` **after** the routes it should cover.

## State

State is cloned per request; use an `Arc`-based type or the `ConnectionPool` directly (it is internally reference-counted):

```rust
async fn get_all_products(
    State(pool): State<ConnectionPool>,
    Query(params): Query<ProductsParams>,
) -> Result<Json<ProductsListResponse>, (StatusCode, String)> {
    // ...
}
```

Install it once with `.with_state(pool)` when building the root router.

## Extractors

- `Path<T>`, `Query<T>`, `Json<T>`, `State<T>`, `Extension<T>` implement `FromRequest`/`FromRequestParts`.
- A handler may take **at most one** `FromRequest` extractor (the body consumer) and it must be **last**:

```rust
async fn patch_product(
    State(pool): State<ConnectionPool>,
    Path(id): Path<Uuid>,
    Json(payload): Json<UpdateProductRequest>, // body consumer — last
) -> Result<Json<MessageResponse>, (StatusCode, String)> { /* ... */ }
```

The repo reads middleware-inserted values with `Extension<AuthUser>` today:

```rust
async fn get_profile(
    State(pool): State<ConnectionPool>,
    Extension(user): Extension<AuthUser>,
) -> Result<Json<DataResponse<Profile>>, (StatusCode, String)> { /* ... */ }
```

If you want authentication to fail earlier (before the body), you can add a `FromRequestParts` impl instead — but note `Extension<AuthUser>` only works because `AuthLayer` always runs first:

```rust
#[async_trait]
impl<S> FromRequestParts<S> for AuthUser
where S: Send + Sync {
    type Rejection = (StatusCode, String);

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        parts.extensions.get::<AuthUser>().cloned()
            .ok_or((StatusCode::UNAUTHORIZED, "Not authenticated".to_string()))
    }
}
```

## Responses

Anything implementing `IntoResponse` can be returned. Common forms:

```rust
Json(value)                                  // 200 + JSON body
(StatusCode::CREATED, Json(value))           // tuple sets status
(StatusCode::NOT_FOUND, "not found")         // Optional<_> not needed: use Result
```

The repo's response envelope is the tuple structs in `routers/categories.rs`:

```rust
#[derive(Serialize)] pub struct MessageResponse { pub mensaje: String }
#[derive(Serialize)] pub struct DataResponse<T: Serialize> { pub datos: T }
```

`api-design` owns the target envelope (`{ "data": ... }` / `{ "error": { "code", "message" } }`); see `architecture.md` §5 for the convergence plan.

## Error handling

Two rejection shapes exist today — match the file you are editing:

```rust
// (StatusCode, String) — categories.rs, products.rs, variants.rs, variations.rs
pub fn internal_error<E: std::error::Error>(err: E) -> (StatusCode, String) {
    (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
}

// (StatusCode, Json<ErrorResponse>) — auth.rs, account.rs, orders.rs, users.rs, reports.rs, directions.rs
#[derive(Serialize)] struct ErrorResponse { error: String }
fn internal_error<E: std::error::Error>(err: E) -> (StatusCode, Json<ErrorResponse>) {
    (StatusCode::INTERNAL_SERVER_ERROR, Json(ErrorResponse { error: err.to_string() }))
}
```

- Validation failures → `(StatusCode::BAD_REQUEST, map_garde_errors(report))`.
- Not found → `(StatusCode::NOT_FOUND, "…".to_string())`.

Both current helpers put the raw error string in the response; `architecture.md` says not to leak internals. Target: `tracing::error!` the real error and return a generic message. Don't widen the leak — fix it when you touch the handler.

## Middleware

Two ways, choose by complexity:

```rust
// Option A — impl Layer + Service (stateful, reusable; used by AuthLayer / RequireRoleLayer)
impl<S> Layer<S> for AuthLayer { /* ... */ }

// Option B — from_fn (simple, per-route logic)
async fn require_admin(
    user: Option<Extension<AuthUser>>,
    request: Request,
    next: Next,
) -> Result<Response, (StatusCode, String)> {
    match user {
        Some(Extension(u)) if u.role >= Role::Admin => Ok(next.run(request).await),
        Some(_) => Err((StatusCode::FORBIDDEN, "Insufficient permissions".to_string())),
        None    => Err((StatusCode::UNAUTHORIZED, "Not authenticated".to_string())),
    }
}

let app = Router::new()
    .route("/api/admin/stats", get(stats))
    .layer(axum::middleware::from_fn(require_admin))
    .layer(AuthLayer::new(secret));
```

Implementing `Layer`/`Service` by hand (as the repo does) is verbose; prefer the `tower` ecosystem layers when possible. See `common/middleware.rs` for the canonical `AuthLayer` + `RequireRoleLayer`.

## Validation with garde

Validate request DTOs before touching the DB:

```rust
#[derive(Validate, Deserialize)]
struct CreateCategoryRequest {
    #[garde(length(min = 1, max = 255))] name: String,
    #[garde(skip)] description: Option<String>,
}

if let Err(report) = payload.validate() {
    return Err((StatusCode::BAD_REQUEST, map_garde_errors(report)));
}
```

## Tower / tower-http layers

Reuse the installed stack instead of hand-rolling: `tower-http::cors`, `tower::timeout::TimeoutLayer`, `tower::retry::RetryLayer`, `tower::limit::{RateLimitLayer, ConcurrencyLimitLayer}` (the worker uses the last two).

## Multipart uploads

`axum-extra` provides `Multipart` (enabled in `Cargo.toml`). Stream fields; enforce size limits before writing to disk (`routers/images.rs`).

## Examples from the field

Generic patterns lifted from `references/axum_middleware.md` and `references/axum_framework_guide.md`.

Auth middleware that bails early with `?` (idiomatic `Result<Response, StatusCode>`):

```rust
use axum::{
    extract::Request,
    http::{StatusCode, header},
    middleware::Next,
    response::Response,
};

async fn require_auth(req: Request, next: Next) -> Result<Response, StatusCode> {
    let token = req
        .headers()
        .get(header::AUTHORIZATION)          // typed constant, not "authorization"
        .and_then(|v| v.to_str().ok())
        .ok_or(StatusCode::UNAUTHORIZED)?;   // bail out before the handler runs
    if token == expected { Ok(next.run(req).await) } else { Err(StatusCode::UNAUTHORIZED) }
}
```

`.layer()` wraps everything including the 404 fallback; `.route_layer()` wraps only defined routes. Prefer `.route_layer()` for auth:

```rust
let app = Router::new()
    .route("/private", get(private_handler))
    .route_layer(from_fn(require_auth));     // 404s never hit auth
```

Layer order is an onion — **the last `.layer()` written is the outermost and runs first**:

```rust
let app = Router::new()
    .route("/dashboard", get(dashboard))
    .route_layer(from_fn(require_auth))  // runs 2nd
    .layer(from_fn(logger));             // runs 1st (outermost = listed last)
```

Custom error type instead of tuple rejections (`?` works in the handler):

```rust
#[derive(Debug)]
enum ApiError { NotFound, Internal(anyhow::Error) }

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let status = match self { ApiError::NotFound => StatusCode::NOT_FOUND, _ => StatusCode::INTERNAL_SERVER_ERROR };
        (status, Json(serde_json::json!({ "error": status.as_str() }))).into_response()
    }
}

async fn handler() -> Result<Json<Item>, ApiError> { /* ... */ }
```

Dynamic path params and nesting use axum 0.8 syntax:

```rust
let items = Router::new().route("/{id}", get(get_item));
let app = Router::new().nest("/api/v1/items", items);
```

## Testing handlers

Test routers without a live server via `tower::ServiceExt::oneshot`:

```rust
use tower::ServiceExt; // for `.oneshot`

let response = app.oneshot(
    Request::builder().uri("/products").body(Body::empty()).unwrap()
).await.unwrap();
assert_eq!(response.status(), StatusCode::OK);
```

See `tests/auth_admin.rs` for an existing integration example.

## Checklist

- [ ] Route paths use `{param}` syntax; layers attached after their routes
- [ ] State passed via `State<T>`; one body extractor, last in the arg list
- [ ] Errors use the rejection shape of the file (`(StatusCode, String)` or `(StatusCode, Json<ErrorResponse>)`)
- [ ] Request DTOs validated with `garde` before DB access
- [ ] Protected routes layered with `AuthLayer` and (where needed) `RequireRoleLayer`
- [ ] No `unwrap()`/`expect()` in handlers (see `architecture.md` "What NOT to do")

## References

- `references/axum_middleware.md` — *Axum Middleware: A Beginner's Complete Guide* (Syeed Talha) — middleware, `from_fn`, layering, layer order.
- `references/axum_framework_guide.md` — *Axum Framework: The Ultimate Guide* (Mastering Backend) — design philosophy, routing, nesting, error handling, REST basics.
