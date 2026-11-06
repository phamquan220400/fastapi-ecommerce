# Implementation Plan: Modular Layered Architecture Refactor

**Branch**: `001-layered-architecture-refactor` | **Date**: 2026-11-06 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/001-layered-architecture-refactor/spec.md`

## Summary

Restructure the backend from global layer folders into business-capability modules under `app/modules/`, with each module containing only the layers justified by its complexity. Use the architecture in `docs/modular-architecture-fastapi.md`: thin API adapters, application use cases and ports, optional framework-independent domain code, and infrastructure adapters. Keep cross-cutting technical concerns focused in `app/core/` and `app/infrastructure/`; preserve the external API, database behavior, and ARQ contracts. Migrate incrementally, with one authoritative implementation per capability.

## Technical Context

**Language/Version**: Python 3.11+ (`pyproject.toml` allows `<4.0`)

**Primary Dependencies**: FastAPI, Pydantic 2, SQLAlchemy 2, Alembic, ARQ; existing Redis, Stripe, Elasticsearch, and observability integrations remain in place. No new dependency is planned.

**Storage**: PostgreSQL via synchronous SQLAlchemy sessions; Redis supports cache, idempotency, and short-lived coordination; Elasticsearch is optional for search.

**Testing**: pytest against a session-scoped PostgreSQL Testcontainers instance; Ruff and mypy; CI also checks migrations against a fresh PostgreSQL database and builds the Docker image.

**Target Platform**: Linux server, with API and ARQ worker as separate processes.

**Project Type**: FastAPI web service with background workers.

**Performance Goals**: No new performance target. Preserve existing behavior and avoid adding avoidable database round trips; benchmark only if an implementation changes the affected operation's performance characteristics.

**Constraints**: No intentional public/admin API, event, database-schema, infrastructure, deployment, or commerce-rule changes. Preserve PostgreSQL locking and transaction semantics, idempotency, outbox atomicity, worker registration/retries, authorization, error behavior, and optional-service outage behavior. Keep Alembic discovery and the migration chain intact. Avoid empty layers and broad shared/common modules.

**Scale/Scope**: Existing commerce capabilities currently spread across `app/api/v1/routes/`, `app/services/`, `app/crud/`, `app/models/`, `app/schema/`, and `app/workers/`, with regression coverage in `tests/`. Candidate modules include shared contracts, users, addresses, categories, products, brands, carts, orders, payments, subscriptions, pricing/currency, loyalty/gift cards/store credit, returns, shipping/tax, promotions/coupons, reviews, wishlists, and notifications/search where ownership warrants separate modules. Final boundaries are confirmed during the inventory phase.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle / gate | Assessment | Plan control |
|---|---|---|
| I. Correctness of Commerce State | Pass with explicit preservation controls | Capture and retain transaction boundaries, row locks, order/payment/inventory state transitions, idempotency, and same-transaction outbox writes; validate success and failure paths with PostgreSQL-backed tests. |
| II. Security at Every Boundary | Pass | Preserve route authentication, authorization, ownership checks, validation, rate limits, and anti-enumeration behavior; do not expose credentials or personal data in new logs/contracts. |
| III. Explicit Boundaries and Simple Design | Pass; current global layout is being replaced | Organize by business capability; use application ports for required cross-module collaboration; keep domain framework-independent; add only layers justified by complexity and avoid catch-all shared packages. |
| IV. Compatible, Migratable Contracts | Pass | Treat generated OpenAPI and existing API/worker behavior as compatibility baselines. No intentional API, event, or database-schema change is planned. |
| V. Observable and Recoverable Operations | Pass | Preserve ARQ task names, registrations, schedules, retry/backoff behavior, and existing logging/metrics/tracing. Document worker idempotency, terminal failure handling, and operator recovery in `quickstart.md`. |
| VI. Verified Change and Sustainable Maintenance | Pass | Run the PostgreSQL-backed suite, Ruff, mypy, dependency audit, fresh-database migration, and Docker gates. Plan new workflow tests as separate tasks from the corresponding code-refactor tasks, as required by FR-007. |
| Infrastructure and simplicity constraints | Pass | No new dependencies, infrastructure, architecture framework, or deployment component. |

No constitution violations require an exception or complexity justification.

## Design and Execution Strategy

1. Establish baselines: generated OpenAPI, worker names/arguments/registration/schedules, model and migration discovery, route-to-persistence access, cross-domain imports, and transaction/lock/side-effect invariants.
2. Define module ownership and dependency order before moving code. Keep `app/main.py` as the composition root; add `app/api/router.py` to aggregate module routers and focused API dependencies only where required.
3. Migrate in bounded dependency-aware slices into `app/modules/<capability>/`. Modules use `api/`, `application/`, `domain/`, and `infrastructure/` when warranted; simpler capabilities may use fewer files/folders, as permitted by the architecture reference.
4. Within each module, API adapters call application use cases; use cases depend on application ports and own orchestration/transaction coordination; domain code contains framework-independent rules; infrastructure implements persistence and external-system adapters. Pydantic API schemas stay in the module boundary and are not used as domain entities.
5. Put database/session setup, unit-of-work implementation if needed, and cross-cutting technical adapters in `app/infrastructure/`; retain focused configuration, errors, logging, and security utilities in `app/core/`. Use `app/modules/shared/` only for small stable business contracts genuinely shared across modules.
6. Route cross-module calls through stable application use cases/ports, never another module's repository or ORM model. Worker tasks remain adapters that invoke application behavior; preserve ARQ names, arguments, registrations, cron schedules, retry logic, and process separation.
7. Migrate model imports and relationships deliberately. Preserve SQLAlchemy metadata registration for Alembic and test setup; do not keep both old and new modules as competing sources of truth.
8. Apply the module boundaries across all in-scope capabilities, including catalog, cart/checkout, orders/payments, accounts/admin, pricing/currency, loyalty/gift cards/store credit, returns/refunds, subscriptions, and related jobs. Keep simple features simple.
9. Plan new unit tests separately from each code-refactoring task per FR-007. Existing tests provide immediate behavior regression protection; new module/use-case unit tests are independent tasks.

## Verification Strategy

- Compare the generated OpenAPI operation set and security metadata before/after; any difference is a blocker unless separately specified and approved.
- Run existing targeted tests for each changed domain, including checkout pricing, guest checkout, idempotency, row-locking/concurrency, outbox processing, subscriptions, returns/refunds, and representative admin flows.
- Run PostgreSQL-backed tests for transactions, row locks, reservations, and migration behavior; mocks or SQLite do not establish those guarantees.
- Run `ruff check app/`, `mypy app/`, the repository's full `pytest` suite, and the dependency audit from the existing CI security job before completion. Preserve the existing fresh-database migration and Docker CI gates.
- Review at least one use case from every migrated business module and record the selected use cases and results in `quickstart.md`. Also trace at least three request-driven workflows (catalog, account, checkout) and two background workflows (subscription billing, outbox processing), recording each result there.
- Ensure no route-level persistence access, domain framework imports, cross-module repository/ORM imports, duplicated authoritative implementations, or circular imports remain in the in-scope paths.

## Project Structure

### Documentation (this feature)

```text
specs/001-layered-architecture-refactor/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
└── contracts/
    └── api-compatibility.md
```

### Source Code (repository root)

```text
app/
├── main.py                         # Composition root
├── api/
│   ├── router.py                   # Aggregate module routers under API version
│   └── deps.py                     # Shared HTTP dependencies, when needed
├── core/                           # Focused config, errors, logging, security utilities
├── infrastructure/                 # Database/session, UoW, observability and technical adapters
├── modules/
│   ├── shared/                     # Only stable shared business contracts, if needed
│   ├── categories/
│   │   ├── api/
│   │   ├── application/
│   │   ├── infrastructure/
│   │   └── schemas.py
│   ├── users/
│   │   └── ...                     # Add domain/use_cases/ports where justified
│   ├── products/
│   │   └── ...
│   ├── orders/
│   │   └── ...
│   └── ...                         # Other commerce capabilities
└── workers/                        # ARQ adapters invoking module application behavior
tests/
├── unit/
├── integration/
└── api/
```

**Structure Decision**: Adopt the module-first layout from `docs/modular-architecture-fastapi.md` within the existing `app` package. The diagram is illustrative, not a requirement to create every directory in every module. Move current routes, schemas, services, CRUD, and ORM files into their owning modules; retain `app/core/` and add a focused `app/infrastructure/` for technical concerns. Keep ARQ in `app/workers/`, update the API aggregation and composition wiring, and preserve model registration for Alembic. Migrate in slices so each capability has one source of truth at every stage.

## Complexity Tracking

No constitution violations; no complexity exception is required.
