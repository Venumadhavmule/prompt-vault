# SKILL 04-C — Production Python API (FastAPI / Django REST Framework)
Category: Backend Development
Applies to: Python, FastAPI, Django, Django REST Framework, Pydantic

## What this skill covers
Python's two dominant API frameworks — FastAPI and Django REST Framework — have different philosophies: FastAPI is async-first with automatic OpenAPI generation; DRF is mature, batteries-included, and synchronous by default. This skill covers how to structure, validate, secure, and deploy both, including when to choose one over the other.

## When to activate this skill
- Starting a new Python REST API
- Adding Pydantic validation to an existing FastAPI app
- Migrating from Flask to FastAPI or DRF
- Choosing between FastAPI and DRF for a new project

## Core principles
1. **Pydantic everywhere in FastAPI.** Use `BaseModel` for request bodies, response schemas, and settings. Never access `request.body` or `request.json()` directly.
2. **DRF serializers are both validation and serialization.** Use them as the single schema layer — never bypass them for raw model access in views.
3. **Async I/O requires an async ORM.** FastAPI endpoints marked `async def` must use `databases`, SQLAlchemy 2 async, or Tortoise ORM — mixing sync ORM calls in async context blocks the event loop.
4. **Settings via `BaseSettings`.** Pydantic `BaseSettings` reads from env vars automatically. Never import `os.environ` inside business logic.
5. **Uvicorn + Gunicorn for production.** `uvicorn --workers` for async; `gunicorn -k uvicorn.workers.UvicornWorker` for multiple processes.

## Step-by-step guide
1. **FastAPI project setup:**
   - `app/main.py` wires router includes and middleware
   - `app/routers/` holds APIRouters by domain
   - `app/schemas/` holds Pydantic request/response models
   - `app/services/` holds business logic
   - `app/db/` holds ORM models and session factory
   - `app/config.py` — a single `Settings` class from `BaseSettings`
2. **DRF project setup:**
   - Use Django's app-per-domain structure: `users/`, `orders/`, `products/`
   - Each app: `models.py`, `serializers.py`, `views.py`, `urls.py`
   - `settings/base.py` + `settings/production.py` + `settings/local.py`
3. **Validate all input:** FastAPI: declare `Request body: MySchema` as a function param. DRF: call `serializer.is_valid(raise_exception=True)`.
4. **Dependency injection in FastAPI:** use `Depends()` for db session, current user, and permission checks — do not use global state.
5. **Custom exception handlers:** in FastAPI, use `@app.exception_handler(MyError)`. In DRF, override `exception_handler` in `settings.REST_FRAMEWORK`.
6. **Middleware:** add CORS (`CORSMiddleware` / `django-cors-headers`), request logging, and rate limiting (`slowapi` / `django-ratelimit`) as middleware.
7. **Production server:** `gunicorn -k uvicorn.workers.UvicornWorker app.main:app --workers 4 --bind 0.0.0.0:8000`

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| `os.environ['SECRET_KEY']` in `models.py` | `Settings` class via `BaseSettings`, injected via `Depends(get_settings)` |
| `async def` route with `session.query()` SQLAlchemy sync call | Use SQLAlchemy 2 `async with AsyncSession` or `databases` for true async |
| Returning ORM objects directly as responses | Always use Pydantic response model (`response_model=MySchema`) to strip internals |
| `serializer.data` before checking `is_valid()` | Always call `serializer.is_valid(raise_exception=True)` first |
| Single `settings.py` file committed with secrets | `python-decouple` or `BaseSettings` reading from `.env`, `.env` in `.gitignore` |

## Stack-specific notes
**FastAPI:** Use `APIRouter` with `prefix` and `tags` to split routes. Automatic OpenAPI docs at `/docs` and `/redoc` in dev. Disable in production with `app = FastAPI(docs_url=None, redoc_url=None)`.
**Django REST Framework:** Use `ModelSerializer` for CRUD, `Serializer` for custom shapes. `ViewSet` + `Router` reduces boilerplate for standard CRUD. Use `django-filter` for filterable list endpoints.
**Node.js / Go (contrast):** Python's synchronous ORM options (Django ORM, SQLAlchemy core) are simpler but block in async contexts. For high-concurrency I/O-heavy APIs, Fastify or Go may be more appropriate.

## Common mistakes
1. **Mixing sync and async in FastAPI.** A `def` route is run in a thread pool; an `async def` route runs in the event loop. Sync ORM calls in `async def` block the loop — use `run_in_executor` or switch to async ORM.
2. **Exposing internal fields.** Without `response_model`, FastAPI returns all model fields including password hashes. Always declare `response_model`.
3. **Not setting `ALLOWED_HOSTS` in Django production.** Django defaults to allowing all hosts. Set `ALLOWED_HOSTS` to your domain(s) in production settings.
4. **Creating a new DB session per request without closing it.** FastAPI's `Depends(get_db)` with a generator pattern opens and closes the session correctly. Ad-hoc session creation leaks connections.

## Checklist
- [ ] Settings managed via `BaseSettings` or `python-decouple` — no `os.environ` calls in business logic
- [ ] All request inputs validated via Pydantic model (FastAPI) or serializer (DRF)
- [ ] Response models declared to strip internal fields
- [ ] Async routes use async ORM (FastAPI async apps)
- [ ] DB session created and closed per request via dependency injection
- [ ] CORS configured with explicit allowed origins (not `*` in production)
- [ ] Custom exception handler returns consistent error format
- [ ] Rate limiting applied on sensitive and expensive endpoints
- [ ] Production server uses Gunicorn + Uvicorn workers
- [ ] OpenAPI docs endpoint disabled in production
- [ ] `ALLOWED_HOSTS` (Django) or equivalent set to known domains in production
