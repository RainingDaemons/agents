# Axum Framework: The Ultimate Guide

> Source: https://blog.masteringbackend.com/axum-framework
> Author: Solomon Eseme (Mastering Backend). Saved as a reference for the `axum-patterns` skill.
>
> **Version note:** several code samples in the original guide use axum 0.6/0.7 conventions
> (e.g. path params as `/:id`, `HandleErrorLayer`, `handle_error`). In axum 0.8 the path
> parameter syntax is `/{id}`, and error handling is done with a custom `IntoResponse` impl
> and/or `tower::ServiceBuilder`. Treat the samples here as concepts; use the 0.8 syntax
> from the skill's `SKILL.md`.

## Design philosophy

Axum is a Tokio-team web framework where handlers are ordinary async functions and request parsing is done by typed extractors. Three core principles:

1. **Simplicity** — a small, intuitive interface for routing and request handling.
2. **Composability** — build applications from small, reusable components.
3. **Scalability** — async foundation handles many concurrent requests with low overhead.

Prerequisites: Rust (ownership, borrowing, `async`/`await`), backend fundamentals (HTTP, REST, CRUD), and a Rust toolchain.

## Built on Tokio

Axum sits on top of the **Tokio** async runtime, inheriting its event-driven handling of async tasks and I/O. Core features:

- High-performance architecture (Rust + Tokio).
- Ergonomic API that does not compromise on usability.
- Composability and integration with the wider Rust ecosystem.
- First-class asynchronous programming via `async`/`await`.

## Routing

Routing maps incoming requests to handlers by URL path and HTTP method, using a declarative syntax.

```rust
use axum::{routing::get, Router};

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(|| async { "Hello, Rustacean!" }));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

### Different HTTP methods

```rust
use axum::{routing::{get, post, put, delete}, Router};

let app = Router::new()
    .route("/", get(root_handler))
    .route("/create", post(create_handler))
    .route("/update", put(update_handler))
    .route("/delete", delete(delete_handler));
```

### Dynamic URL parameters

```rust
use axum::{routing::get, Router, extract::Path};

async fn user_handler(Path(user_id): Path<u32>) -> String {
    format!("User ID: {}", user_id)
}

// axum 0.8: "/user/{user_id}"
let app = Router::new().route("/user/{user_id}", get(user_handler));
```

### Nested routes and scoping

```rust
use axum::{routing::get, Router};

let users_routes = Router::new().route("/list", get(users_list));
let posts_routes = Router::new().route("/all", get(posts_all));

let app = Router::new()
    .nest("/users", users_routes)
    .nest("/posts", posts_routes);
```

### Middleware in routes

```rust
use axum::{Router, routing::get, middleware::from_fn};

let app = Router::new()
    .route("/", get(root_handler))
    .layer(from_fn(auth_middleware));
```

### Prefixes and route aggregation

```rust
use axum::{Router, routing::get};

let api_routes = Router::new().route("/v1/items", get(items_handler));
let app = Router::new().nest("/api", api_routes);
```

### Query extractors

```rust
use axum::{Router, routing::get, extract::Query, Json};
use std::collections::HashMap;
use serde_json::Value;

async fn search_handler(Query(params): Query<HashMap<String, String>>) -> Json<Value> {
    // Process query parameters...
    Json(Value::Null)
}

let app = Router::new().route("/search", get(search_handler));
```

## Middleware

Middleware sits between the incoming request and the handler. It can process the request, alter the response, log, and enforce auth. In axum it is a Tower layer attached with `.layer()` or `.route_layer()`.

```rust
use axum::{middleware::from_fn, Router, routing::get};

async fn logging_middleware(req: Request, next: Next) -> Response {
    println!("Request received: {:?}", req.uri());
    next.run(req).await
}

let app = Router::new()
    .route("/", get(root_handler))
    .layer(from_fn(logging_middleware));
```

Middleware can be chained, and can perform async work (DB queries, external calls). See `references/axum_middleware.md` for the full middleware guide.

## Error handling

Handlers can return `Result<T, E>` where `E: IntoResponse`, so Rust's `?` operator composes naturally. Define a custom error type and implement `IntoResponse`:

```rust
use axum::{http::StatusCode, response::IntoResponse};

#[derive(Debug)]
struct MyError { message: String }

impl MyError {
    fn new(message: &str) -> Self { MyError { message: message.to_string() } }
}

impl IntoResponse for MyError {
    fn into_response(self) -> axum::response::Response {
        (StatusCode::INTERNAL_SERVER_ERROR, self.message).into_response()
    }
}

async fn handler() -> Result<String, MyError> {
    Err(MyError::new("An error occurred"))
}

let app = axum::Router::new().route("/", axum::routing::get(handler));
```

Concepts from the original guide (older axum APIs): error propagation up the middleware stack, and a global error handler layered over the app. In axum 0.8 prefer a custom `IntoResponse` error enum for the same effect; `tower::ServiceBuilder` + `HandleErrorLayer` is the modern equivalent for service-layer errors.

Best practices: descriptive messages, log errors, degrade gracefully.

## Building RESTful APIs

REST endpoints map to resources; HTTP methods map to operations (GET read, POST create, PUT/PATCH update, DELETE delete).

```rust
use axum::{routing::{get, post, put, delete}, Router, Json};

let app = Router::new()
    .route("/users", get(list_users).post(create_user))
    .route("/users/{id}", get(get_user).put(update_user).delete(delete_user));
```

### Database connectivity

The guide uses SQLx with a `PgPool`:

```rust
use sqlx::postgres::PgPoolOptions;

let database_url = std::env::var("DATABASE_URL").expect("DATABASE_URL must be set");
let pool = PgPoolOptions::new()
    .connect(&database_url)
    .await
    .expect("Failed to connect to database");
```

A handler reading rows, injecting the pool via `Extension` (concept only — this repo uses `State` + bb8):

```rust
use axum::{Json, Extension};
use sqlx::PgPool;

#[derive(serde::Serialize, serde::Deserialize, sqlx::FromRow)]
pub struct Task { pub id: i32, pub title: String, pub completed: bool }

async fn list_tasks(Extension(pool): Extension<PgPool>) -> Json<Vec<Task>> {
    let tasks = sqlx::query_as::<_, Task>("SELECT * FROM tasks")
        .fetch_all(&pool)
        .await
        .expect("Failed to fetch tasks");
    Json(tasks)
}
```

A transactional write:

```rust
use axum::http::StatusCode;
use sqlx::PgPool;

async fn create_task(Json(task): Json<Task>, Extension(pool): Extension<PgPool>) -> Result<Json<Task>, StatusCode> {
    let mut tx = pool.begin().await.map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    let task = sqlx::query_as::<_, Task>(
        "INSERT INTO tasks (title, completed) VALUES ($1, $2) RETURNING id, title, completed",
    )
    .bind(&task.title)
    .bind(task.completed)
    .fetch_one(&mut *tx)
    .await
    .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    tx.commit().await.map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    Ok(Json(task))
}
```

## Conclusion

Axum brings async programming to Rust backend development: concise routing, middleware support, `IntoResponse`-based error handling, and seamless Tokio integration. It suits REST APIs, real-time applications, and microservices.
