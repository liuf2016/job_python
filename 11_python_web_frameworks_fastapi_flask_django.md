# Session 11 — Python Web Frameworks: FastAPI, Flask, Django (+ Microservices & MCP Servers)

**Time:** Concepts 15 · Quiz 10 · Interview 15
Focus: basic concepts, a key-feature summary of each framework, and interview questions. Prerequisites: Sessions 2 (decorators), 4 (async), 5 (typing).

> **The question you were asked:** *"Should endpoint handlers all live in one file, or be spread out by functionality?"*
> **Answer: spread out by functionality (feature/domain).** One file per resource or domain (users, orders, auth), each exposing a router, assembled in a thin `main.py`. See 11.4 for the full reasoning and a layout you can sketch on a whiteboard.

---

## Part 1 — Concepts

### 11.1 Web basics you must be able to explain
- **HTTP**: request = method + URL + headers + body; response = status + headers + body. Methods: `GET` (read, safe, idempotent), `POST` (create/non-idempotent), `PUT` (replace, idempotent), `PATCH` (partial update), `DELETE` (idempotent).
- **Status codes**: 200 OK, 201 Created, 204 No Content, 400 Bad Request, 401 Unauthenticated, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable (FastAPI validation error), 429 Too Many Requests, 500 Server Error, 502/503/504 gateway/unavailable/timeout.
- **REST conventions**: nouns not verbs (`/orders/42`), plural collections, status codes carry meaning, JSON bodies, pagination (`?limit=&cursor=`), versioning (`/v1/`), idempotency keys for safe retries on POST.
- **WSGI vs ASGI**: WSGI (Flask, classic Django) is a synchronous interface: one request per worker thread/process. ASGI (FastAPI/Starlette, modern Django) supports async, WebSockets, streaming, long-lived connections.
- **Serving**: framework app ← server (Gunicorn for WSGI; **Uvicorn**/Hypercorn for ASGI; commonly `gunicorn -k uvicorn.workers.UvicornWorker` or `uvicorn --workers N`) ← reverse proxy (Nginx/ALB) ← client. Never use the dev server (`flask run`, `runserver`, `uvicorn --reload`) in production.

### 11.2 Framework landscape at a glance

| | **FastAPI** | **Flask** | **Django** |
|---|---|---|---|
| Style | Modern, async-first, API-focused | Micro-framework: minimal core, you pick the parts | "Batteries included" full-stack |
| Protocol | ASGI (built on Starlette) | WSGI (async support is limited) | WSGI + ASGI |
| Validation | Built-in via **Pydantic** + type hints | Manual / extensions (marshmallow, pydantic) | Forms/serializers (DRF serializers) |
| Docs | **Auto OpenAPI + Swagger UI (`/docs`)** | Extension (flask-smorest, apispec) | DRF + drf-spectacular |
| DB / ORM | None built in (SQLAlchemy, SQLModel, Tortoise) | None built in (Flask-SQLAlchemy) | **Built-in ORM**, migrations, admin |
| Auth | Tools (OAuth2, security utils); you wire it | Flask-Login, extensions | Built-in users, sessions, permissions |
| Templates/HTML | Possible (Jinja2) but not the focus | Jinja2 built in | Django templates built in |
| Best for | JSON APIs, microservices, ML model serving, MCP/AI backends | Small services, prototypes, flexibility | Content-heavy sites, admin-driven apps, monoliths, fast CRUD with auth |
| Typical API add-on | — | Flask-RESTX/smorest | **Django REST Framework (DRF)** / Django Ninja |

Rule of thumb: **new JSON API/microservice → FastAPI. Full product with admin, auth, ORM, server-rendered pages → Django. Small/legacy/simple, or you want total control → Flask.**

### 11.3 FastAPI essentials

```python
# main.py
from fastapi import FastAPI, HTTPException, Depends, Query, status
from pydantic import BaseModel, EmailStr, Field

app = FastAPI(title="Shop API", version="1.0")

class UserCreate(BaseModel):                    # request body schema (validated)
    email: EmailStr
    name: str = Field(min_length=1, max_length=80)

class UserOut(BaseModel):                       # response schema (filters fields, e.g. no password)
    id: int
    email: EmailStr
    name: str

@app.post("/users", response_model=UserOut, status_code=status.HTTP_201_CREATED)
async def create_user(payload: UserCreate, svc: "UserService" = Depends(get_user_service)):
    return await svc.create(payload)

@app.get("/users/{user_id}", response_model=UserOut)       # path parameter (typed: int)
async def get_user(user_id: int, svc=Depends(get_user_service)):
    user = await svc.get(user_id)
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return user

@app.get("/users", response_model=list[UserOut])
async def list_users(limit: int = Query(20, le=100), offset: int = 0):   # query parameters
    ...
```
Run: `uvicorn main:app --reload` → interactive docs at `/docs` (Swagger) and `/redoc`; schema at `/openapi.json`.

**Key features**
- **Type hints drive everything**: parameter source is inferred (path if in the URL template, query for simple types, body for Pydantic models; `Header()`, `Cookie()`, `Form()`, `File()` explicit). Invalid input → automatic **422** with field-level errors. No hand-written validation.
- **Pydantic models** for request/response; `response_model` serializes and filters output. Pydantic v2 core is Rust-based (fast). Use separate models for *Create / Update / Out / DB* (never return the DB model or password hash directly).
- **Dependency injection** (`Depends`): reusable, composable, overridable in tests. Used for DB sessions, current user, pagination params, settings, auth.
  ```python
  async def get_db():
      async with SessionLocal() as session:
          yield session                       # yield-dependency: cleanup after response

  async def current_user(token: str = Depends(oauth2_scheme), db=Depends(get_db)) -> User: ...

  @app.get("/me")
  async def me(user: User = Depends(current_user)): return user
  ```
- **`async def` vs `def` endpoints**: `async def` runs on the event loop — use only with non-blocking (awaitable) libraries; a blocking call (`requests`, sync DB driver, `time.sleep`) **freezes the whole server**. Plain `def` endpoints are run in a threadpool automatically — safe for blocking code. Choose by what your calls are (Session 4).
- **Routers** (`APIRouter`): modular grouping of endpoints (11.4).
- **Middleware** (CORS, GZip, request-ID/logging, timing), **exception handlers** (`@app.exception_handler`), **lifespan** events (startup/shutdown: create connection pools, load ML models):
  ```python
  from contextlib import asynccontextmanager
  @asynccontextmanager
  async def lifespan(app: FastAPI):
      app.state.pool = await create_pool()      # startup
      yield
      await app.state.pool.close()              # shutdown
  app = FastAPI(lifespan=lifespan)
  ```
- **Background work**: `BackgroundTasks` for small post-response tasks (send email). For durable/heavy jobs use a queue (Celery, RQ, Arq, Dramatiq) — background tasks die with the process and have no retries.
- **Other**: WebSockets, streaming responses / Server-Sent Events, file upload, OAuth2/JWT helpers, `Settings` via `pydantic-settings` (env vars), `TestClient`/`httpx.AsyncClient`, `app.dependency_overrides` for testing.
- **Under the hood**: FastAPI = **Starlette** (ASGI toolkit: routing, middleware, requests) + **Pydantic** (data validation) + OpenAPI generation.

### 11.4 Project structure: why NOT one file (the question you were asked)

**One giant `main.py` problems:** merge conflicts across a team; hard to navigate/test/review; circular imports and tangled globals; cannot own, version, or deploy features independently; cannot reuse or extract into a service later; violates single responsibility.

**Recommended: split by domain/feature, with layers inside each feature.**

```
app/
├── main.py                 # create app, include routers, middleware, lifespan  (thin)
├── core/
│   ├── config.py           # Settings (pydantic-settings), env vars
│   ├── security.py         # password hashing, JWT
│   └── logging.py
├── db/
│   ├── session.py          # engine, session factory, get_db dependency
│   └── base.py
├── api/
│   ├── deps.py             # shared dependencies (current_user, pagination)
│   └── v1/
│       ├── router.py       # aggregates feature routers under /v1
│       ├── users.py        # HTTP layer only: parse → call service → respond
│       ├── orders.py
│       └── auth.py
├── models/                 # SQLAlchemy ORM models (DB tables)
├── schemas/                # Pydantic models (request/response contracts)
├── services/               # business logic (framework-agnostic, easy to unit test)
├── repositories/           # data access (SQL queries) — optional layer
└── tests/
```
Alternative (also valid, preferred in larger systems): **package-per-feature** — `app/users/{router,schemas,models,service}.py`, `app/orders/{...}` — keeps everything for one feature together (mirrors "modular monolith" and eases later extraction into microservices). Either way: *organize by domain*, not by dumping unrelated endpoints together.

```python
# app/api/v1/users.py
from fastapi import APIRouter, Depends
router = APIRouter(prefix="/users", tags=["users"])

@router.get("/{user_id}", response_model=UserOut)
async def get_user(user_id: int, svc: UserService = Depends(get_user_service)):
    return await svc.get_or_404(user_id)

# app/main.py
app = FastAPI(lifespan=lifespan)
app.include_router(users.router, prefix="/v1")
app.include_router(orders.router, prefix="/v1")
```
Equivalents: Flask **Blueprints** (`bp = Blueprint("users", __name__)` + `app.register_blueprint(bp, url_prefix="/users")`; use an **application factory** `create_app()`), Django **apps** (`python manage.py startapp users`, each with `models/views/urls`, wired in the root `urls.py` via `include()`).

**Principles to cite:** separation of concerns (HTTP layer is thin → services hold business rules → repositories talk to DB), single responsibility, high cohesion/low coupling, dependency injection for testability, versioned routes, and keep handlers free of SQL and business logic.
**Nuance to show maturity:** a tiny prototype or a 3-endpoint service can start in one file — but design for splitting early (routers from day one is nearly free) and split before it becomes painful. Don't over-engineer with five layers for a trivial CRUD app.

### 11.5 Flask essentials
```python
from flask import Flask, jsonify, request, abort, Blueprint

def create_app(config=None):                    # application factory: testable, multiple configs
    app = Flask(__name__)
    app.config.from_object(config or "config.Dev")
    app.register_blueprint(users_bp, url_prefix="/users")
    return app

users_bp = Blueprint("users", __name__)

@users_bp.get("/<int:user_id>")
def get_user(user_id):
    user = db.get(user_id) or abort(404)
    return jsonify(user.to_dict())

@users_bp.post("/")
def create_user():
    data = request.get_json()                   # no automatic validation (add pydantic/marshmallow)
    ...
    return jsonify(user.to_dict()), 201
```
- **Core**: Werkzeug (WSGI utilities) + Jinja2 (templates). Everything else is an extension: Flask-SQLAlchemy, Flask-Migrate, Flask-Login, Flask-CORS, Flask-JWT-Extended.
- **Context locals**: `request`, `g`, `session`, `current_app` are *proxies* bound per-request (thread/context-local). Convenient; a gotcha outside a request context (`RuntimeError: Working outside of request context`).
- Hooks: `before_request`, `after_request`, `errorhandler`. Config via `app.config` / env vars.
- Strengths: simple, flexible, huge ecosystem, easy to learn. Weaknesses: no built-in validation/docs/async-first design; structure and consistency depend on discipline (hence the factory + blueprints advice).
- Async: `async def` views work but run in a per-request event loop (not true ASGI concurrency); use Quart or FastAPI for async-heavy services.

### 11.6 Django essentials
- **MTV** (Model–Template–View; ≈ MVC): `models.py` (ORM), views (controllers: function or class-based), templates (presentation), `urls.py` (routing).
  ```python
  # models.py
  class Order(models.Model):
      customer = models.ForeignKey(Customer, on_delete=models.CASCADE, related_name="orders")
      total = models.DecimalField(max_digits=10, decimal_places=2)
      created_at = models.DateTimeField(auto_now_add=True)

  # ORM queries are lazy QuerySets
  Order.objects.filter(total__gt=100).select_related("customer").order_by("-created_at")[:20]
  ```
- **Batteries included**: ORM + **migrations** (`makemigrations`, `migrate`), **admin site** (instant back-office UI), authentication/sessions/permissions, forms, CSRF/XSS/SQL-injection protections by default, caching framework, signals, management commands, i18n, test client.
- **Middleware** chain; **settings.py** per environment; **apps** as reusable modules; **class-based views**.
- **Django REST Framework (DRF)** for APIs: `Serializer`/`ModelSerializer` (validation + (de)serialization), `APIView`, `ViewSet` + `Router` (CRUD in a few lines), permissions, authentication classes, pagination, throttling, filtering, browsable API.
  ```python
  class OrderViewSet(viewsets.ModelViewSet):
      queryset = Order.objects.select_related("customer")
      serializer_class = OrderSerializer
      permission_classes = [IsAuthenticated]
  router = DefaultRouter(); router.register("orders", OrderViewSet)
  ```
- **Performance classics**: **N+1 queries** → `select_related` (FK/one-to-one, SQL JOIN) and `prefetch_related` (many-to-many/reverse FK, second query); `only()/defer()`, `bulk_create`, DB indexes (`db_index`, `Meta.indexes`), `transaction.atomic()`, `F()` expressions for atomic updates, `django-debug-toolbar` to spot query counts.
- Async: ASGI support, async views and increasingly async ORM methods (`aget`, `async for`), but many parts still sync (thread-sensitive adaptors). Alternative: **Django Ninja** (FastAPI-like on Django).

### 11.7 Cross-cutting API concerns (framework-independent, always asked)
- **Authentication vs authorization**: who you are (401) vs what you may do (403). Mechanisms: session cookies (browser apps; CSRF protection needed), **JWT** bearer tokens (stateless; short expiry + refresh tokens; can't revoke easily), **OAuth2/OIDC** (delegated auth), API keys (service-to-service). Hash passwords with bcrypt/argon2, never store plain. Use HTTPS.
- **Validation at the edge**, consistent error format (e.g. RFC 9457 problem details: `type`, `title`, `status`, `detail`).
- **Pagination**: offset (simple, slow deep pages) vs cursor/keyset (stable and fast).
- **Idempotency & retries**, rate limiting (Redis token bucket / API gateway), timeouts everywhere, CORS configuration (not `*` with credentials).
- **Caching**: HTTP caching (ETag/Cache-Control), Redis for hot data; invalidate carefully.
- **Observability**: structured logs with correlation/request IDs, metrics (Prometheus), tracing (OpenTelemetry), `/health` (liveness) and `/ready` (readiness) endpoints.
- **Database sessions**: one session per request via dependency; commit/rollback explicitly; connection pools sized to `workers × pool_size ≤ DB max_connections`.
- **Concurrency/scaling**: stateless app processes (no in-memory user state) → scale horizontally behind a load balancer; state in DB/Redis.
- **Security basics (OWASP)**: injection (parameterized queries), broken auth, excessive data exposure (response models), mass assignment (separate input schemas), SSRF, secrets in env/secret manager, dependency scanning.
- **Testing**: unit test services; API tests via `TestClient` (FastAPI) / `app.test_client()` (Flask) / Django `Client`; override dependencies; isolated DB per test (transaction rollback or test containers).
- **API versioning & compatibility**: additive changes are non-breaking; breaking changes → new version; OpenAPI as the contract; contract tests.

### 11.8 Microservices (the concepts interviewers probe)
- **Monolith vs modular monolith vs microservices.** Start with a well-structured (modular) monolith; split into services when team size, scaling needs, or deploy independence justify the cost. Microservices trade code complexity for operational/distributed-system complexity.
- Each service **owns its data** (database per service); no shared DB tables. Communicate via:
  - **Synchronous**: REST/HTTP or gRPC — simple, but couples availability (use timeouts, retries with backoff + jitter, **circuit breakers**, bulkheads).
  - **Asynchronous**: message brokers/event streams (RabbitMQ, Kafka, SQS) — decoupled, resilient, eventual consistency.
- **Distributed transactions**: no cross-service ACID → **Saga** pattern (compensating actions), **outbox pattern** (write event + data in one local transaction, publish reliably), idempotent consumers.
- **API gateway** (routing, auth, rate-limit), **service discovery**, config/secrets management, containerization (Docker) + orchestration (Kubernetes), CI/CD per service, distributed tracing.
- **Failure thinking**: network is unreliable, partial failure is normal, design for retries and idempotency; CAP/eventual consistency awareness.
- Python specifics: FastAPI is a popular choice (small, fast to build, typed contracts); share schemas via OpenAPI/generated clients or a shared contracts package (careful with coupling); pin per-service dependencies; one Docker image per service; health endpoints; graceful shutdown on SIGTERM.

### 11.9 MCP servers (Model Context Protocol) in Python
- **MCP** is an open protocol that lets AI applications (Claude, IDE agents) connect to external **tools**, **resources**, and **prompts** through a standard client–server interface (JSON-RPC 2.0 messages).
  - **Tools**: functions the model can call (side effects allowed): "search tickets", "run query".
  - **Resources**: read-only data identified by URI (files, DB rows, docs).
  - **Prompts**: reusable prompt templates.
  - **Transports**: `stdio` (local subprocess launched by the client — simplest, common for desktop/CLI) and **Streamable HTTP** (remote servers; older SSE transport is legacy). Remote servers typically add OAuth-based authorization.
- Minimal server with the official Python SDK (`pip install "mcp[cli]"`; the higher-level `FastMCP` API; there is also a separately packaged `fastmcp` library with similar decorators):
  ```python
  from mcp.server.fastmcp import FastMCP

  mcp = FastMCP("orders")

  @mcp.tool()
  def get_order_status(order_id: int) -> str:
      """Return the status of an order."""             # docstring + type hints become the tool schema
      return lookup(order_id).status

  @mcp.resource("orders://{order_id}")
  def order_doc(order_id: int) -> str: ...

  if __name__ == "__main__":
      mcp.run()                                         # stdio by default; HTTP transports configurable
  ```
- Why it fits this lesson: the pattern is the same as FastAPI — **type hints + docstrings generate the schema**, handlers are small functions, validation comes from the types. An MCP server can wrap your existing service layer; many teams expose the same business logic via REST (for apps) and MCP (for agents).
- Design guidance: small, well-named tools with clear descriptions and strict input schemas; return concise, structured results; least privilege and confirmation for destructive actions; validate and sanitize inputs (the model's arguments are untrusted input — **prompt injection** and confused-deputy risks); auth for remote servers; log tool calls; keep tools idempotent when possible; never write to `stdout` in stdio servers except protocol messages (log to stderr).
- Also seen: FastAPI apps mounting an MCP server, or auto-generating MCP tools from an OpenAPI spec (e.g., `FastMCP.from_fastapi`-style tooling).

---

## Part 2 — Homework / Exercises (not timed)

1. **Refactor**: take a single-file FastAPI app with `users`, `orders`, and `auth` endpoints (write it first, ~100 lines, with an in-memory dict as the DB) and split it into the layout in 11.4: routers, schemas, services, `deps.py`, `main.py`. Verify `/docs` still shows all endpoints grouped by tag.
2. **Dependency injection**: create a `get_current_user` dependency (bearer token → user), use it on `/me`, and write a pytest that overrides it with `app.dependency_overrides` to test protected endpoints without real auth.
3. **Validation**: add `Field` constraints and a custom `@field_validator`; call the API with bad data and read the 422 response body.
4. **Sync vs async**: write one endpoint with `async def` calling `time.sleep(2)` and another with `def` calling `time.sleep(2)`. Fire 10 concurrent requests (`hey`/`httpx`/`asyncio.gather`) at each and explain the timing difference.
5. **Database**: wire SQLAlchemy 2.0 async (or sync + `def` endpoints) with a per-request session dependency; implement CRUD for one resource with pagination; return a 409 on duplicate email.
6. **Same API three ways**: implement `GET/POST /notes` in Flask (with a Blueprint + factory), Django + DRF (`ModelViewSet`), and FastAPI; compare lines of code, validation, and docs.
7. **Django ORM**: create `Author`/`Book` models; write a view listing books with author names that causes N+1; fix with `select_related`; confirm query count with `assertNumQueries` or debug toolbar.
8. **Error handling**: add a global exception handler returning a consistent JSON error body, plus a request-ID middleware that adds `X-Request-ID` to responses and logs.
9. **MCP**: build a stdio MCP server with two tools (`add_note`, `search_notes`) and a resource; test it with the MCP Inspector (`mcp dev server.py`) or by registering it in Claude Desktop/Claude Code (`claude mcp add`).
10. **Whiteboard**: draw how you'd split an e-commerce monolith into services (users, catalog, orders, payments); mark which calls are sync vs events and where a saga is needed.

<details><summary>Solution hints</summary>

1. Typical split: `schemas/user.py` (Pydantic), `services/user_service.py` (logic, no FastAPI imports), `api/v1/users.py` (router), `api/deps.py` (`get_user_service`, `current_user`). A service that doesn't import FastAPI is the sign you split correctly.
4. The `async def` + `time.sleep` version serializes: ~20 s for 10 requests (blocks the loop). The `def` version runs in the threadpool (default ~40 threads): ~2 s. With `await asyncio.sleep(2)` the async version is also ~2 s.
5. Catch `IntegrityError` → `HTTPException(409)`; or check first, but rely on the DB unique constraint to avoid races.
7. Without `select_related("author")`: 1 + N queries. With it: 1 JOIN query.
</details>

---

## Part 3 — Quiz

**Q1.** Handlers for users, orders, and auth in one 2,000-line `main.py` vs separate modules — which and why (give 3 reasons)?

**Q2.** What does FastAPI return when a request body fails Pydantic validation? Did you write that code?

**Q3.** Where do `APIRouter` (FastAPI), `Blueprint` (Flask), and an "app" (Django) fit? One-line equivalence.

**Q4.** In FastAPI, what happens if an `async def` endpoint calls `requests.get(...)`?

**Q5.** WSGI vs ASGI — what is the practical difference?

**Q6.** What is `Depends()` used for? Give 3 typical uses.

**Q7.** Why define separate Pydantic models for `UserCreate` and `UserOut`?

**Q8.** What problem do `select_related` and `prefetch_related` solve in Django?

**Q9.** What does `BackgroundTasks` do, and when is it the wrong tool?

**Q10.** 401 vs 403?

**Q11.** MCP: what are tools, resources, and prompts? Which transport would you use for a local desktop integration?

**Q12.** A service in your microservice system calls another over HTTP and the callee is slow. Name three protections.

**Q13.** Which framework would you choose for: (a) ML inference API, (b) internal admin + CRUD website with login, (c) a 5-endpoint webhook receiver you want to write in 20 minutes?

<details><summary>Answers</summary>

1. Separate modules (routers/blueprints/apps by feature). Reasons: fewer merge conflicts and easier parallel work; navigability, review, and testability (import only what you test); separation of concerns/low coupling (business logic out of the HTTP layer); independent evolution and a path to extract a service; avoids circular imports and global tangles. Caveat: a tiny prototype can begin in one file, but use routers early.
2. `422 Unprocessable Entity` with a JSON list of errors (`loc`, `msg`, `type`). No — automatic from type hints and Pydantic.
3. `APIRouter` ≈ `Blueprint` ≈ a Django app's `urls.py` included via `include()` — modular groups of routes mounted into the main application.
4. It blocks the event loop; every other request on that worker stalls until it returns. Use an async client (`httpx.AsyncClient`), or declare the endpoint with `def` (threadpool), or `await asyncio.to_thread(...)`.
5. WSGI is synchronous request/response, one request per worker at a time, no WebSockets. ASGI supports async I/O, WebSockets, streaming, and many concurrent connections per worker.
6. Dependency injection: DB session per request, current user/auth, pagination/query params, settings, rate limiting, service/repository objects. Overridable in tests.
7. Input and output differ: create has a password and no `id`; output has `id` and hides secrets. Prevents mass assignment and accidental data leakage; clearer API contract/docs.
8. N+1 queries. `select_related` uses a SQL JOIN for FK/one-to-one; `prefetch_related` runs one extra query and joins in Python for many-to-many/reverse relations.
9. Runs a function after the response is sent, in the same process. Wrong for heavy, long-running, or must-not-lose work (no persistence/retry; dies on restart) — use Celery/RQ/Arq/Dramatiq or a queue.
10. 401: not authenticated (missing/invalid credentials). 403: authenticated but not permitted.
11. Tools: model-invoked functions (actions). Resources: read-only addressable data. Prompts: reusable templates. Local: `stdio`; remote: Streamable HTTP.
12. Timeouts, retries with backoff + jitter (only for idempotent calls), circuit breaker, bulkheads/concurrency limits, fallback/cached response, async messaging instead of sync call.
13. (a) FastAPI (async, typed, easy to load model in lifespan; heavy CPU work should be in worker processes/batching). (b) Django (admin, ORM, auth, templates). (c) Flask or FastAPI — both fine; FastAPI if you want validation/docs for free.
</details>

---

## Part 4 — Interview Questions

**I1. Should all endpoint handlers go in one file, or be organized by functionality? (The one you were asked.)**
<details><summary>Model answer</summary>
"Organize by functionality (domain), not one file. In FastAPI I create one `APIRouter` per resource — `users.py`, `orders.py`, `auth.py` — each with its own prefix and tags, and `main.py` just creates the app and includes the routers. That gives smaller files, fewer merge conflicts, easier testing, clear ownership, and a clean path to pulling a domain out into its own service. I also keep handlers thin: parse and validate input, call a service layer, return a response model. Business logic lives in services and SQL in repositories, with shared things like DB sessions and the current user provided through `Depends`. Flask's equivalent is Blueprints with an application factory, and Django's is separate apps wired with `include()`. For a tiny prototype one file is acceptable, but I'd still use routers from day one because it costs almost nothing. I'd avoid over-engineering, though — don't add four layers to a three-endpoint service." Likely follow-ups: *by layer or by feature?* (feature-first scales better; layers inside each feature), *where do shared models go?* (a `core`/`common` package; avoid cross-feature imports that create cycles), *how do you version?* (`/v1` router prefix).
</details>

**I2. Why is FastAPI popular? What does it give you over Flask?**
<details><summary>Model answer</summary>
Type hints become validation, serialization, and documentation: Pydantic models validate input and shape output, and OpenAPI/Swagger docs are generated automatically. Built on ASGI/Starlette, so it handles async I/O, WebSockets, and streaming; DI via `Depends`; very good performance for a Python framework (limited by Python and your I/O, not magic); excellent editor support. Flask is simpler and more flexible, but validation, docs, and async need extensions or manual work. Trade-offs: younger ecosystem than Django/Flask, no built-in ORM/admin/auth, and async pitfalls (blocking the loop).
</details>

**I3. Explain `async def` vs `def` endpoints in FastAPI and the biggest pitfall.**
<details><summary>Model answer</summary>
`async def` runs on the event loop, so it must only `await` non-blocking operations; `def` endpoints run in a threadpool, so blocking calls are OK. The pitfall: blocking code (sync DB driver, `requests`, CPU-heavy work, `time.sleep`) inside `async def` freezes all requests on that worker. Fix by using async libraries (asyncpg, httpx), `asyncio.to_thread`, or a plain `def`; offload CPU-bound work to processes/queue workers. Scale with multiple Uvicorn workers/processes.
</details>

**I4. How does dependency injection work in FastAPI and why use it?**
<details><summary>Model answer</summary>
Parameters annotated with `Depends(callable)` cause FastAPI to call the dependency (resolving its own dependencies recursively, caching per request), and pass the result in. `yield` dependencies run cleanup after the response (close DB session). Benefits: reuse (auth, pagination, DB), separation of concerns, and testability via `app.dependency_overrides` to swap real DB/auth for fakes. Compare with Python's general DI-by-parameters from Session 5.
</details>

**I5. How do you design request/response models and avoid leaking data?**
<details><summary>Model answer</summary>
Separate schemas per use: `UserCreate` (with password), `UserUpdate` (all optional), `UserOut` (no secrets), and the ORM model for storage. Set `response_model` (or return type) so the output is filtered; `from_attributes=True` (Pydantic v2) to read ORM objects. Prevents mass assignment and over-exposure (OWASP). Validate with `Field` constraints and validators; use enums/Literals for allowed values.
</details>

**I6. Flask vs Django vs FastAPI — how do you choose?**
<details><summary>Model answer</summary>
Use the comparison table: FastAPI for new typed JSON APIs, microservices, async/I/O-heavy and ML/AI backends; Django when you need a full product quickly — ORM, migrations, admin, auth, forms, templates — plus DRF for APIs; Flask for small apps/prototypes or when you want to assemble your own stack. Also consider team experience, existing codebase, hiring, ecosystem maturity, and operational needs. Say that you've seen/can use all three and that the concepts (routing, middleware, validation, ORM sessions, auth) transfer.
</details>

**I7. How would you implement authentication/authorization for an API?**
<details><summary>Model answer</summary>
Passwords hashed with argon2/bcrypt; login issues a short-lived JWT access token (and a refresh token), sent as `Authorization: Bearer`; a `current_user` dependency/middleware decodes and verifies (signature, expiry, audience), loads the user, and raises 401 on failure. Authorization via roles/scopes/permissions checked in dependencies (403). For third-party login use OAuth2/OIDC with an identity provider rather than rolling your own. Considerations: HTTPS only, token revocation strategy, secrets in env/secret manager, rate-limiting login, CORS, and CSRF if you use cookies.
</details>

**I8. Your API is slow under load. How do you investigate?**
<details><summary>Model answer</summary>
Measure first: tracing/APM and structured logs with request IDs; look at p50/p95/p99 latency per endpoint; load test (locust/k6). Common causes: N+1 queries or missing indexes (`EXPLAIN ANALYZE`), blocking calls in async handlers, too few workers or exhausted DB connection pool, serialization overhead of huge payloads (pagination), no caching, chatty calls to other services (batch, parallelize with `asyncio.gather`, add timeouts). Fixes: indexes, eager loading, caching (Redis), async I/O, background queue for slow work, horizontal scaling (stateless workers), and streaming/pagination for large responses.
</details>

**I9. What is a microservice, and when would you not use microservices?**
<details><summary>Model answer</summary>
An independently deployable service around a business capability that owns its data and communicates via APIs/events. Benefits: independent scaling/deploys, team autonomy, fault isolation, technology freedom. Costs: distributed-system complexity (network failures, eventual consistency, observability, versioning, testing, ops overhead). Don't start there for a small team or unclear domain boundaries — begin with a modular monolith (feature-based routers/packages) and extract services when there's a real scaling or ownership need. Mention saga/outbox, idempotency, circuit breakers, contract testing, and API gateways.
</details>

**I10. How do services communicate, and how do you keep data consistent across them?**
<details><summary>Model answer</summary>
Sync: REST/gRPC (simple, request/response; needs timeouts, retries, circuit breakers). Async: events/messages via Kafka/RabbitMQ/SQS (decoupled; at-least-once delivery means consumers must be idempotent). Consistency: no distributed ACID by default → saga (orchestrated or choreographed, with compensating actions), outbox pattern to publish events atomically with DB writes, and accept eventual consistency where the business allows; use idempotency keys for retried commands.
</details>

**I11. What is MCP and how would you build an MCP server in Python?**
<details><summary>Model answer</summary>
Model Context Protocol: an open standard (JSON-RPC based) for exposing tools, resources, and prompts to AI clients so they can act on external systems without bespoke integrations. In Python use the official SDK's `FastMCP`: decorate functions with `@mcp.tool()` / `@mcp.resource()`; type hints and docstrings generate the JSON schema the model sees; run over stdio for local use or Streamable HTTP for remote (with auth). Design: narrow tools, clear descriptions, validated inputs, least privilege, human confirmation for destructive actions, protection against prompt injection, logging to stderr (not stdout) on stdio servers. Relationship to REST: both can sit on the same service layer — REST for apps, MCP for agents.
</details>

**I12. How would you test a FastAPI service?**
<details><summary>Model answer</summary>
Pyramid: unit tests for services (no framework); API tests with `TestClient` (or `httpx.AsyncClient` + `pytest-asyncio`) exercising routes, validation, status codes; override dependencies (`dependency_overrides`) for auth and DB; real Postgres via testcontainers/docker-compose for integration with per-test transaction rollback; contract tests against the OpenAPI schema; load tests separately; CI runs lint, type-check, tests. Test error paths (422, 404, 409, 401/403), not just happy paths.
</details>

**I13 (Django). Explain the request lifecycle in Django, and how you'd fix N+1 queries.**
<details><summary>Model answer</summary>
Request → WSGI/ASGI handler → middleware (request phase) → URL resolver → view (function or class-based; DRF viewset/serializer for APIs) → ORM/DB and templates/serializers → response → middleware (response phase) → client. N+1: view loops over objects and accesses a related field per row; fix with `select_related` (JOIN) / `prefetch_related` (extra query), verify with `assertNumQueries` or debug toolbar, add DB indexes, paginate, use `only()`/`values()` for large reads.
</details>

**I14 (Flask). What are the application factory and request context in Flask?**
<details><summary>Model answer</summary>
Factory: `create_app(config)` builds and configures the app instance, registers blueprints/extensions — enables multiple configs, avoids circular imports, makes testing easy (fresh app per test). Request/app context: `request`, `g`, `current_app` are context-bound proxies that only work while a context is active (inside a request, or `with app.app_context():` in scripts/tests); implemented with `contextvars`.
</details>

**Whiteboard drill (10 min):** "Design a REST API for a to-do app with users and shared lists." Cover resources and URLs (`/v1/lists/{id}/items`), methods and status codes, auth, schemas (Create/Update/Out), pagination, error format, file/project layout (routers + services + repositories), DB tables and indexes, and how you'd later extract "notifications" into its own service.
