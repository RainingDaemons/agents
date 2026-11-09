# Axum Middleware: A Beginner's Complete Guide

> Source: https://dev.to/syeedmdtalha/axum-middleware-a-beginners-complete-guide-1p6a
> Author: Syeed Talha. Updated for Axum 0.8.x. Saved as a reference for the `axum-patterns` skill.

## 1. What is middleware?

Middleware is code that runs between the moment a request arrives at your server and the moment your handler deals with it. It runs on the way **in** (before the handler) and on the way **out** (after the handler).

```
// Without middleware
Client  ──────────────────────────────>  Handler

// With middleware
Client  ->  Logger  ->  Auth  ->  Handler
        <-          <-        <-
        (response flows back the same way)
```

Common uses: logging, authentication, rate limiting, CORS headers, compression, request IDs.

## 2. Why middleware

Without it, per-route concerns (auth, logging) get copy-pasted into every handler. Middleware writes that logic once and applies it to many routes.

## 3. How Axum does middleware

Axum is built on **Tower**. Middleware in Axum is a Tower `Layer`. You normally do not implement Tower traits by hand — use the `from_fn` adapter:

```rust
use axum::middleware::from_fn;

// Your middleware is just an async fn
async fn my_middleware(req: Request, next: Next) -> Response {
    // ... your logic here
}

let app = Router::new()
    .route("/hello", get(handler))
    .layer(from_fn(my_middleware));
```

## 4. Correct imports (Axum 0.8)

`Request` is generic over the body type. Axum 0.8 ships a pre-aliased version; import it from `axum::extract`, never `axum::http`:

```rust
// ✅ CORRECT — pre-aliased, body type filled in
use axum::extract::Request;
use axum::middleware::Next;
use axum::response::Response;

// ❌ WRONG — raw generic, compiler complains
use axum::http::Request;
```

Full import block:

```rust
use axum::{
    extract::Request,            // ← axum::extract, NOT axum::http
    middleware::{from_fn, Next},
    http::{StatusCode, header},  // header::AUTHORIZATION etc.
    response::{Response, IntoResponse},
};
```

## 5. Function signature

```rust
async fn my_middleware(req: Request, next: Next) -> Response {
    // body
}
```

- `req` — incoming `axum::extract::Request`. Gives `req.method()`, `req.uri()`, `req.headers()`, `req.extensions()`.
- `next` — everything after this middleware (the rest of the stack, then the handler). It is a "proceed" button: call `next.run(req).await` to pass forward. If you never call it, the request stops here (useful to reject early).
- `-> Response` — must always return a response. Two ways:
  - let the handler produce it: `next.run(req).await`
  - reject early: `(StatusCode::UNAUTHORIZED, "bad token").into_response()`

## 6. The two zones

```rust
use axum::{extract::Request, middleware::Next, response::Response};

async fn my_middleware(req: Request, next: Next) -> Response {
    // ── ZONE 1: runs BEFORE the handler ──
    println!("→ incoming: {} {}", req.method(), req.uri());

    let response = next.run(req).await;  // <-- THE PIVOT POINT

    // ── ZONE 2: runs AFTER the handler ──
    println!("← outgoing: {}", response.status());

    response
}
```

## 7. Logging requests

Easy way with `tower-http`:

```toml
[dependencies]
axum        = "0.8"
tokio       = { version = "1", features = ["full"] }
tower-http  = { version = "0.6", features = ["trace"] }
tracing     = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

```rust
use tower_http::trace::TraceLayer;
use tracing_subscriber::EnvFilter;

tracing_subscriber::fmt()
    .with_env_filter(
        EnvFilter::try_from_default_env()
            .unwrap_or_else(|_| EnvFilter::new("axum_tracing_example=info,tower_http=trace")),
    )
    .init();

let app = Router::new()
    .route("/users", get(get_users))
    .layer(TraceLayer::new_for_http());
```

Manual logger when you want full control:

```rust
use axum::{extract::Request, middleware::Next, response::Response};
use std::time::Instant;

async fn logger(req: Request, next: Next) -> Response {
    let method = req.method().clone();
    let uri    = req.uri().clone();
    let start  = Instant::now();

    let response = next.run(req).await;

    println!(
        "{} {} {} {}ms",
        method, uri, response.status(), start.elapsed().as_millis()
    );
    response
}
```

## 8. Auth middleware

The official docs recommend returning `Result<Response, StatusCode>` — it allows the `?` operator for early bail-out:

```rust
use axum::{
    extract::Request,
    http::{StatusCode, header},
    middleware::Next,
    response::Response,
};

async fn require_auth(req: Request, next: Next) -> Result<Response, StatusCode> {
    let auth_header = req
        .headers()
        .get(header::AUTHORIZATION)
        .and_then(|v| v.to_str().ok())
        .ok_or(StatusCode::UNAUTHORIZED)?;

    if auth_header == "Bearer my-secret-token" {
        Ok(next.run(req).await)
    } else {
        Err(StatusCode::UNAUTHORIZED)
    }
}
```

### `.layer()` vs `.route_layer()`

```rust
// .layer() — wraps EVERYTHING, including the 404 fallback
let app = Router::new()
    .route("/private", get(private_handler))
    .layer(from_fn(require_auth));

// .route_layer() — wraps only defined routes, NOT the fallback (preferred for auth)
let app = Router::new()
    .route("/private", get(private_handler))
    .route_layer(from_fn(require_auth));
```

## 9. Stacking middlewares

Layers wrap like an onion. **The last `.layer()` written is the outermost and runs first on incoming requests.**

```rust
let app = Router::new()
    .route("/dashboard", get(dashboard))
    .route_layer(from_fn(require_auth))  // runs 2nd
    .layer(from_fn(logger));             // runs 1st (outermost = listed last)
```

```
Request comes in
     |
     v
[ logger middleware ]           <- outermost, runs first
  [ require_auth middleware ]   <- inner, runs second
    [ your handler ]            <- innermost, runs last
  [ require_auth middleware ]   <- on the way back out
[ logger middleware ]           <- on the way back out
     |
     v
Response goes out
```

## 10. Complete working example

```toml
[dependencies]
axum               = "0.8"
tokio              = { version = "1", features = ["full"] }
serde              = { version = "1", features = ["derive"] }
tower-http         = { version = "0.6", features = ["trace"] }
tracing-subscriber = "0.3"
```

```rust
use axum::{
    extract::Request,
    http::{StatusCode, header},
    middleware::{self, Next},
    response::{Json, Response},
    routing::get,
    Router,
};
use serde::Serialize;
use std::time::Instant;
use tower_http::trace::TraceLayer;

async fn logger(req: Request, next: Next) -> Response {
    let method = req.method().clone();
    let uri    = req.uri().clone();
    let start  = Instant::now();
    let response = next.run(req).await;
    println!("{} {} → {} ({}ms)", method, uri, response.status(), start.elapsed().as_millis());
    response
}

async fn require_auth(req: Request, next: Next) -> Result<Response, StatusCode> {
    let token = req
        .headers()
        .get(header::AUTHORIZATION)
        .and_then(|v| v.to_str().ok())
        .ok_or(StatusCode::UNAUTHORIZED)?;
    if token == "Bearer secret123" { Ok(next.run(req).await) } else { Err(StatusCode::UNAUTHORIZED) }
}

#[derive(Serialize)]
struct Message { text: String }

async fn public_route() -> Json<Message> { Json(Message { text: "Anyone can see this".into() }) }
async fn private_route() -> Json<Message> { Json(Message { text: "You are authenticated!".into() }) }

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt::init();

    let app = Router::new()
        .route("/public",  get(public_route))
        .route("/private", get(private_route))
        .route_layer(middleware::from_fn(require_auth)) // auth: route-level only
        .layer(middleware::from_fn(logger))             // logger: global
        .layer(TraceLayer::new_for_http());

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

## 11. Common beginner mistakes

```rust
// 1. Wrong Request import
use axum::http::Request;   // ❌  →  use axum::extract::Request;  ✅

// 2. The semicolon trap — drops the value
async fn bad(req: Request, next: Next) -> Response { next.run(req).await; } // ❌

// 3. Moving `req` before you are done with it
let method = req.method();            // borrows req
let response = next.run(req).await;   // moves req — compiler error
// fix: let method = req.method().clone();

// 4. Wrong layer order — last written runs first
.layer(from_fn(logger))
.layer(from_fn(auth))   // ❌ this runs first, not logger
```

## 12. Quick reference

| You want to… | Use |
|---|---|
| Import `Request` correctly | `use axum::extract::Request` |
| Log all requests easily | `TraceLayer::new_for_http()` (`tower-http 0.6`) |
| Write custom middleware | `from_fn(your_async_fn)` |
| Read request method | `req.method().clone()` |
| Read request path | `req.uri().clone()` |
| Read the Authorization header | `req.headers().get(header::AUTHORIZATION)` |
| Pass request to handler | `next.run(req).await` |
| Reject early (`-> Response`) | `return (StatusCode::UNAUTHORIZED, "msg").into_response()` |
| Reject early (`-> Result`) | `return Err(StatusCode::UNAUTHORIZED)` |
| Wrap specific routes only | `.route_layer(from_fn(...))` |
| Wrap everything incl. fallback | `.layer(from_fn(...))` |

## Summary

- Middleware runs between the request arriving and the handler running.
- `use axum::extract::Request` in Axum 0.8, never `axum::http`.
- `from_fn` turns a plain `async fn` into a middleware layer.
- `-> Result<Response, StatusCode>` is the idiomatic auth-middleware return type.
- `next: Next` is the proceed button; `next.run(req).await` passes forward.
- `.route_layer()` wraps only routes; `.layer()` wraps everything including the 404 fallback.
- The last `.layer()` written is the outermost and runs first.
