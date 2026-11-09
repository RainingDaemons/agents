---
name: python-backend-design
description: Architecture and data-access patterns for the CDN service — FastAPI app lifespan, psycopg async pool, the auth dependency, pydantic-settings config, static file serving, upload validation, and rate limiting. Use when adding CDN endpoints, wiring the DB pool, config, or upload handling.
license: MIT
metadata:
  author: rainingdaemons/agents
  version: "1.0"
---

# Backend Patterns (FastAPI)

How this backend is structured and how to add to it without fighting its design. It covers the **architecture and data access** of this service; FastAPI framework mechanics are in `fastapi`, Python language idioms in `python-patterns`, HTTP contract design in `api-design`, and SQL/schema review in `postgresql-patterns` / `postgresql-optimization` / `database-design`.

## When to activate

- Adding or changing a endpoint in `app/main.py`
- Touching the database pool, auth dependency, config, or upload handling
- Reviewing the backend structure, error handling, or rate limiting

## Request flow

```
HTTP → FastAPI router → Depends(require_admin) [auth + RBAC] → handler
```

- Public endpoints: `GET /images/{filename}`, `GET /health`.
- Mutation endpoints (`POST`, `PUT`, `DELETE` `/images`) require the admin role.
- Rate limiting runs per-IP inside mutation handlers.
- Errors are `HTTPException(status_code, detail)`.
- Handlers are thin: validate → act → respond. Keep business logic in the handler; no service/repository layer.

## App structure & lifespan

The pool is created and closed via the FastAPI lifespan, not per request:

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    await init_pool()
    logger.info("CDN started — serving %s on %s:%s", settings.public_dir, settings.host, settings.port)
    yield
    await close_pool()

app = FastAPI(title="SLEG CDN", lifespan=lifespan)
```

The static mount is registered **last** so API routes match first:

```python
app.mount("/", StaticFiles(directory=settings.public_dir, html=True), name="static")
```

## Database access (psycopg, no ORM)

One module-level async pool in `app/database.py`; create it in the lifespan, borrow connections with `async with`:

```python
pool: psycopg_pool.AsyncConnectionPool | None = None

async def init_pool() -> None:  # AsyncConnectionPool(conninfo=settings.dsn, min_size=1, max_size=5, timeout=10, open=False); await pool.open()
async def close_pool() -> None: ...
def get_pool() -> psycopg_pool.AsyncConnectionPool:
    if pool is None:
        raise RuntimeError("Database pool not initialized")
    return pool
```

```python
db = get_pool()
async with db.connection() as conn:
    await conn.execute("SELECT 1")
```

- No ORM. Do not add SQLAlchemy — use the psycopg pool.
- Use parameterized queries (`%s` placeholders); never format values into SQL.
- The CDN only reads shared tables (schema is owned by `database/migrations/`); do not manage migrations here.

## Auth dependency

`app/auth.py` implements `require_admin` as a FastAPI dependency. It decodes an **HS256 JWT** (signed with `settings.jwt_secret`) and checks the `role` claim:

```python
@dataclass
class AdminUser:
    user_id: UUID

async def require_admin(
    credentials: Annotated[HTTPAuthorizationCredentials | None, Depends(_bearer_scheme)],
) -> AdminUser:
    if credentials is None:
        raise HTTPException(status_code=401, detail="Missing authorization")
    try:
        payload = jwt.decode(credentials.credentials, settings.jwt_secret, algorithms=["HS256"])
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")
    if payload.get("role", "") != "admin":
        raise HTTPException(status_code=403, detail="Admin access required")
    return AdminUser(user_id=UUID(payload.get("sub") or payload.get("user_id")))
```

Attach it per route; never re-implement auth checks in handlers:

```python
@app.delete("/images/{filename}")
async def delete_image(
    filename: str,
    admin: Annotated[AdminUser, Depends(require_admin)],
    request: Request,
):
    ...
```

> Doc drift: `agentwork/docs/architecture.md` describes an opaque-token lookup against `login_tokens`; the code decodes a JWT. Treat the code as the source of truth and fix the doc when you touch auth.

## Config (pydantic-settings)

`app/config.py` declares one `Settings(BaseSettings)` read from `.env`; expose derived values as properties:

```python
class Settings(BaseSettings):
    host: str = "0.0.0.0"
    public_dir: str = "./public"
    jwt_secret: str = ""
    max_upload_size_mb: int = 10
    allowed_image_types: str = "image/jpeg,image/png,image/webp"

    @property
    def max_upload_bytes(self) -> int:
        return self.max_upload_size_mb * 1024 * 1024

    @property
    def dsn(self) -> str: ...

    model_config = {"env_file": ".env", "extra": "ignore"}

settings = Settings()
```

- Read secrets from the environment via `settings`; never hardcode them.
- Import the singleton: `from .config import settings`.

## Errors

- Raise `HTTPException(status_code=..., detail="...")` with a meaningful status (see `api-design` for status semantics).
- Log unexpected errors with `logging.error()` at the handler boundary.
- Never leak internals (DB errors, stack traces) in `detail`.

## Static files & uploads

Serving and writing files must stay inside `public/`:

- **Path traversal:** `validate_filename()` rejects `\x00`, `/`, `\`, `..`, and names not matching `^[a-zA-Z0-9][a-zA-Z0-9._\-]*$`. `_resolve_image()` re-checks the resolved path starts with `settings.public_path`.
- **Magic bytes:** validate uploads by content (`detect_mime_from_bytes` via `puremagic` + `validate_image_contents` via Pillow), never by the `Content-Type` header alone; reject mismatched extensions with 415.
- **Atomic writes:** write to a temp file in `public/`, then `os.replace()`; clean up the temp file on failure.
- **Size limit:** reject bodies over `settings.max_upload_bytes` (413) before and after reading.

```python
fd, tmp_path = tempfile.mkstemp(dir=str(settings.public_path), suffix=".tmp")
try:
    with os.fdopen(fd, "wb") as f:
        f.write(contents)
    os.replace(tmp_path, str(target_path))
except Exception:
    if os.path.exists(tmp_path):
        os.unlink(tmp_path)
    raise
```

## Rate limiting (mechanism)

In-memory per-IP sliding window on mutation endpoints (30 requests / 60 s). Keep the policy in one helper; the HTTP semantics and headers are `api-design`'s concern:

```python
_rate_buckets: dict[str, list[float]] = defaultdict(list)

def _check_rate_limit(key: str) -> None:
    now = time.monotonic()
    _rate_buckets[key] = [t for t in _rate_buckets[key] if now - t < _RATE_LIMIT_WINDOW]
    if len(_rate_buckets[key]) >= _RATE_LIMIT_MAX:
        raise HTTPException(status_code=429, detail="Rate limit exceeded")
    _rate_buckets[key].append(now)
```

Call it inside the handler: `_check_rate_limit(request.client.host if request.client else "unknown")`.

## Logging

- Use the `logging` module (`logging.getLogger("cdn")`), never `print()`.
- Configure once (`logging.basicConfig(level=logging.INFO, ...)`).
- Log lifecycle events (startup), warnings, and unexpected errors; do not log tokens or secrets.

## Testing

`pytest` + `pytest-asyncio`, driving the app with `httpx.AsyncClient` over `ASGITransport`; mock the pool and settings to isolate DB calls. Tests live in `tests/`, named `test_<action>_<scenario>`. See `cdn/agentwork/docs/conventions.md` for the exact commands.

## Module layout

```
app/main.py       FastAPI app, lifespan, routes, rate limit, upload/static logic
app/config.py     Settings (pydantic-settings) + derived properties
app/database.py   AsyncConnectionPool lifecycle, magic-byte detection, MIME maps
app/auth.py       AdminUser, require_admin, validate_filename
public/           served static files (images)
tests/            pytest + httpx tests
```

## Checklist

- [ ] Pool created/closed in the lifespan; handlers borrow via `get_pool()`
- [ ] No ORM; parameterized psycopg queries only
- [ ] Mutations depend on `require_admin`; auth not re-implemented in handlers
- [ ] Config via `settings`; no hardcoded secrets
- [ ] Errors are `HTTPException`; internals logged, not returned
- [ ] Filenames validated and resolved inside `public/`; uploads validated by magic bytes; writes atomic
- [ ] Mutation endpoints rate-limited via `_check_rate_limit`
- [ ] `logging`, not `print`; no tokens/secrets in logs
