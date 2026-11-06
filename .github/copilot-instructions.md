# Copilot Instructions for fastapi-ecommerce

## What this is

A FastAPI + SQLAlchemy 2.x e-commerce backend: catalog, cart/checkout, Stripe
payments and subscriptions, multi-currency pricing, loyalty points/gift
cards, guest checkout, returns/refunds, and an observability stack
(Prometheus/Loki/Tempo). Dependency management is Poetry (`requirements.txt`
is kept in sync only for Docker builds — don't hand-edit it; regenerate with
`poetry export`).

## Build, run, test

```bash
poetry install --with dev        # install deps (CI's source of truth)
poetry run alembic upgrade head  # apply migrations (bootstraps schema from empty)
poetry run uvicorn app.main:app --reload
arq app.workers.arq_worker.WorkerSettings   # background worker (separate process)
```

Or via `make install|run|migrate|test|lint|format` (see [Makefile](/mnt/c/Users/Admin/Documents/DATN/Makefile)).

**Tests** run against a real, ephemeral PostgreSQL container via
testcontainers (Docker must be running) — not SQLite — because the app
relies on Postgres-only behavior (`func.strftime` would be a SQLite-ism to
avoid; `with_for_update(skip_locked=True)` row locking needs real Postgres
semantics). No `DATABASE_URL` is needed to run tests.

```bash
pytest                                   # full suite
pytest tests/test_orders.py              # one file
pytest tests/test_orders.py::test_name   # one test
pytest -k "idempotency"                  # by keyword
ruff check app/                          # lint (CI command)
mypy app/                                # type-check (CI command)
```

The Postgres container is session-scoped (one container for the whole run);
each test gets isolation via `TRUNCATE ... RESTART IDENTITY CASCADE` in the
`db_session` fixture in [tests/conftest.py](/mnt/c/Users/Admin/Documents/DATN/tests/conftest.py), not a fresh
container. The `client` fixture overrides `get_db`, mocks `redis_client`
methods (`get_json`/`set_json`/`delete`/`delete_pattern`/the raw client), and
disables the rate limiter — write new route tests against this fixture
rather than re-mocking Redis yourself.

CI ([.github/workflows/ci.yml](/mnt/c/Users/Admin/Documents/DATN/.github/workflows/ci.yml)) has separate jobs for lint+mypy, pytest,
`pip-audit`, an `alembic upgrade head` run against a brand-new empty
Postgres database (catches migrations that assume pre-existing schema), and
a Docker build that verifies the image runs as a non-root user.

## Architecture: layered, by domain

Each domain (order, product, cart, subscription, ...) follows the same
four-layer stack, wired together in [app/dependencies.py](/mnt/c/Users/Admin/Documents/DATN/app/dependencies.py):

```
app/api/v1/routes/<domain>.py  → FastAPI routers, request/response schemas, auth deps
app/services/<domain>_service.py → business logic, orchestrates CRUD + other services
app/crud/<domain>.py            → SQLAlchemy queries, no business logic
app/models/<domain>.py          → SQLAlchemy ORM models
app/schema/<domain>_schema.py   → Pydantic request/response schemas
```

Routers get a service instance via a `get_<domain>_service_dep` factory in
`app/dependencies.py` (e.g. `get_order_service_dep`), which itself depends on
`get_db` (a per-request `Session`) and sometimes `get_redis_manager`. New
domains should add both a service class and a corresponding `_dep` factory
there rather than instantiating services directly in routes.

Routes are registered in [app/api/v1/init_routes.py](/mnt/c/Users/Admin/Documents/DATN/app/api/v1/init_routes.py). Most routers declare
their own prefix at `app.include_router(...)`, but some set `prefix=` inside
the router module itself (e.g. `tax_rate.router`, `shipping.admin_router`) —
check the router file before adding a prefix at the include-site, or you'll
double it up (see the comment above the `tax_rate` include).

All new models must be imported in [app/models/__init__.py](/mnt/c/Users/Admin/Documents/DATN/app/models/__init__.py) — this is what
makes Alembic autogenerate and `Base.metadata.create_all` (used by the test
suite) discover them. Schema is otherwise managed exclusively by Alembic
migrations; nothing in app code should call `Base.metadata.create_all()`.

## Key conventions

- **Row-level locking for stock**: order placement and variant/product stock
  checks use `select(...).with_for_update()` (see [app/crud/order.py](/mnt/c/Users/Admin/Documents/DATN/app/crud/order.py)); the
  reservation-cleanup worker loop uses `with_for_update(skip_locked=True)` so
  multiple worker instances can poll the same table concurrently without
  blocking each other. Follow this pattern for anything else that decrements
  shared inventory/counters.
- **Outbox pattern**: business transactions that need a side effect (e.g. an
  order being placed) write an `OutboxEvent` row in the same DB transaction
  instead of calling out directly. A periodic ARQ cron job
  (`process_outbox_events_task` in [app/workers/arq_worker.py](/mnt/c/Users/Admin/Documents/DATN/app/workers/arq_worker.py)) picks up
  pending events, retries failures with exponential backoff up to
  `MAX_OUTBOX_ATTEMPTS`, and a separate job prunes old completed rows. There
  is no real message broker wired up — `_publish_event` only logs — so swap
  that one function out when one exists; nothing else needs to change.
- **Background jobs run in a separate ARQ worker**, not the API process.
  Register new tasks in both `WorkerSettings.functions` (for
  `arq_pool.enqueue_job(...)`) and `WorkerSettings.cron_jobs` (for scheduled
  tasks) at the bottom of `app/workers/arq_worker.py` — a function that's
  only defined but never added to `functions`/`cron_jobs` silently never
  runs (this happened historically with the outbox processor).
- **Idempotency**: payment/order-creating endpoints accept an
  `Idempotency-Key` header via the `check_idempotency` dependency
  ([app/utils/idempotency.py](/mnt/c/Users/Admin/Documents/DATN/app/utils/idempotency.py)); on a cache hit it raises
  `IdempotentResponseException`, caught by a custom exception handler in
  `app/main.py` to replay the original response. On success, call
  `cache_idempotent_response(idempotency_key, ...)` before returning. The
  Redis key is hashed from `Authorization header + path + Idempotency-Key`
  to prevent cross-user collisions.
- **Multi-currency**: every price is stored in `BASE_CURRENCY_CODE` (see
  `app/core/config.py` and `app/models/currency.py`); conversion to a
  customer-facing currency happens at display/checkout time, not storage
  time. Loyalty points similarly convert through
  `POINTS_EARNED_PER_BASE_CURRENCY_UNIT` / `POINTS_REDEMPTION_VALUE` — see
  `PaymentService._handle_successful_payment` and
  `calculate_points_discount` in [app/services/pricing.py](/mnt/c/Users/Admin/Documents/DATN/app/services/pricing.py).
- **Naive UTC datetimes**: use `utcnow()` from [app/utils/time.py](/mnt/c/Users/Admin/Documents/DATN/app/utils/time.py) — not
  `datetime.utcnow()` (deprecated) or `datetime.now(timezone.utc)` directly —
  both as a plain call and as a SQLAlchemy column
  `default=utcnow`/`onupdate=utcnow`. Every `DateTime` column in this app is
  a naive `timestamp without time zone`, so an aware datetime would mismatch.
- **SQLAlchemy forward refs in models**: `app/models/*.py` is excluded from
  ruff's `F821` and has `name-defined`/`no-redef` disabled in mypy, because
  `Mapped["X"]` / `relationship("X", ...)` string refs are resolved by the
  ORM registry at runtime, and the `@x.expression` decorator on
  `hybrid_property` legitimately redefines an instance method — not real
  bugs, don't "fix" these in models/.
- **Lifespan-gated infra connections**: `app/main.py`'s `lifespan` detects
  `"pytest" in sys.modules` and swaps Redis/ARQ/cache for mocks/in-memory
  backends instead of connecting to real infra — keep this check if you add
  new startup-time infra connections, or the test suite will try to dial out.
  Elasticsearch connects in a background task (not awaited at startup) so a
  slow/unavailable ES doesn't block the app from serving requests; routes
  that use it must already tolerate a `None` client.
- **Naming convention object**: `app/db/database.py` defines an explicit
  SQLAlchemy `naming_convention` (`ix_`, `uq_`, `ck_`, `fk_`, `pk_` prefixes)
  on `Base.metadata` — Alembic autogenerate relies on this for stable
  constraint names across migrations; don't bypass it with inline
  `Index`/`UniqueConstraint` names.
- **Rate limiting**: endpoints that should be throttled use
  `@limiter.limit("N/minute")` from `app.core.limiter` (slowapi); the
  `client` test fixture disables it globally, so rate-limit behavior itself
  needs its own targeted test rather than being exercised incidentally.
