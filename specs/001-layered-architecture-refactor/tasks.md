---

description: "Implementation tasks for the modular layered architecture refactor"
---

# Tasks: Modular Layered Architecture Refactor

**Input**: Design documents from `specs/001-layered-architecture-refactor/`

**Prerequisites**: `plan.md`, `spec.md`, `research.md`, `data-model.md`, `contracts/`, `quickstart.md`

**Tests**: New module/use-case unit tests are listed separately from the refactoring tasks as required by FR-007. Existing PostgreSQL-backed tests remain the regression suite for API and persistence behavior.

## Phase 1: Setup

**Purpose**: Capture compatibility baselines and decide module ownership before moving code.

- [X] T001 Record the pre-refactor OpenAPI document, including paths, methods, schemas, statuses, and security metadata, in `specs/001-layered-architecture-refactor/contracts/openapi-baseline.json`.
- [X] T002 Inventory current route-to-persistence access, cross-domain imports, transaction/locking invariants, model discovery, and ARQ task names/schedules, retry/idempotency behavior, terminal failure handling, and existing operator recovery procedures; update the ownership and dependency map in `specs/001-layered-architecture-refactor/research.md`.

---

## Phase 2: Foundational

**Purpose**: Establish the module package boundary and stable shared contracts that the capability slices can build on.

- [ ] T003 Create the `app/modules/` package scaffold in `app/modules/__init__.py` and add only the stable cross-module business contracts identified in `specs/001-layered-architecture-refactor/research.md` under `app/modules/shared/`.

**Checkpoint**: Module ownership and cross-module contracts are defined; capability migration can begin.

---

## Phase 3: User Story 1 - Work within a business module (Priority: P1) 🎯 MVP

**Goal**: Place in-scope business responsibilities in capability-owned modules with inward dependencies and no direct access to another module's persistence internals.

**Independent Test**: Trace representative catalog, checkout, and account workflows and verify their API, application, optional domain, schema, and infrastructure responsibilities live under their owning `app/modules/<capability>/` directories. Verify modules use application-level contracts rather than another module's repositories or ORM models, and no rule has competing authoritative copies.

### Unit tests for User Story 1

- [ ] T004 [P] [US1] Add fake-port unit tests for catalog and account use cases, including validation and ownership failures, in `tests/unit/modules/test_catalog_accounts.py`.
- [ ] T005 [P] [US1] Add fake-port unit tests for cart and pricing use cases, including calculation and failure paths, in `tests/unit/modules/test_cart_pricing.py`.
- [ ] T006 [P] [US1] Add fake-port unit tests for order, payment, inventory, and return use cases, including idempotency and state-transition failures, in `tests/unit/modules/test_orders_payments_inventory_returns.py`.
- [ ] T007 [P] [US1] Add fake-port unit tests for subscription, loyalty, and stored-value use cases, including renewal and reversal boundaries, in `tests/unit/modules/test_subscriptions_loyalty.py`.
- [ ] T008 [P] [US1] Add fake-port unit tests for review, wishlist, and customer-engagement use cases in `tests/unit/modules/test_customer_engagement.py`.

### Implementation for User Story 1

- [ ] T009 [US1] Move account and catalog routes, schemas, application behavior, and persistence ownership into `app/modules/users/`, `app/modules/addresses/`, `app/modules/categories/`, `app/modules/brands/`, and `app/modules/products/`; preserve contracts and use application ports for cross-module calls.
- [ ] T010 [US1] Move cart, checkout, currency, tax, shipping, coupon, and promotion behavior into the capability modules selected in `specs/001-layered-architecture-refactor/research.md`, using `app/modules/carts/` and `app/modules/pricing/` as the target roots; keep checkout orchestration in application use cases and calculations in their owning capabilities.
- [ ] T011 [US1] Move order, payment, inventory, return, and refund routes, schemas, use cases, and persistence adapters into their owning `app/modules/` directories; preserve transaction boundaries, PostgreSQL row locks, idempotency, and same-transaction outbox writes.
- [ ] T012 [US1] Move subscription, loyalty, gift-card, and store-credit behavior into their owning `app/modules/` directories; route cross-module operations through application-level ports and preserve base-currency accounting and reversible order snapshots.
- [ ] T013 [US1] Move reviews, wishlists, product questions/answers, notification subscriptions, and search-related capability behavior into their owning `app/modules/` directories; keep optional-service failure behavior and avoid empty layers.

**Checkpoint**: The selected capability modules own their behavior and pass their new unit tests independently.

---

## Phase 4: User Story 2 - Use existing commerce capabilities (Priority: P1)

**Goal**: Keep public and administrative API contracts, authorization outcomes, persistence behavior, and commerce state transitions unchanged while the new module structure is wired into the application.

**Independent Test**: Compare generated OpenAPI against `specs/001-layered-architecture-refactor/contracts/openapi-baseline.json` and run the existing representative PostgreSQL-backed success and failure workflows; verify no unexpected contract or persisted-state differences.

### Tests for User Story 2

- [ ] T014 [US2] Add an API compatibility test comparing registered paths, methods, request/response schemas, status codes, and security metadata with `specs/001-layered-architecture-refactor/contracts/openapi-baseline.json` in `tests/test_api_compatibility.py`.

### Implementation for User Story 2

- [ ] T015 [US2] Aggregate module routers and preserve API version prefixes and route ordering in `app/api/router.py`; wire the aggregate router from `app/main.py` without changing the registered HTTP surface.
- [ ] T016 [US2] Update dependency composition for module application services in `app/dependencies.py` and `app/api/deps.py`, preserving request-scoped database sessions, Redis behavior, authentication, authorization, and rate limiting.
- [ ] T017 [US2] Preserve SQLAlchemy model and relationship discovery for Alembic and test metadata creation by updating `app/models/__init__.py` and `alembic/env.py` to import the module-owned mapped classes exactly once.
- [ ] T018 [US2] Replace imports of legacy global route, service, CRUD, model, and schema implementations with module API/application contracts throughout `app/` and `tests/`; remove obsolete duplicate implementations after callers are migrated.
- [ ] T019 [US2] Validate the existing PostgreSQL-backed representative API flows in `specs/001-layered-architecture-refactor/quickstart.md`, including guest checkout, pricing, idempotency, concurrency, subscriptions, returns, refunds, and administrative behavior; resolve any changed response or state outcome in the owning module.

**Checkpoint**: The API and database contracts match their captured baseline and representative commerce flows pass.

---

## Phase 5: User Story 3 - Operate background commerce workflows (Priority: P2)

**Goal**: Keep ARQ workers as adapters to module application behavior while retaining task interfaces, schedules, retries, and operational failure visibility.

**Independent Test**: Verify representative worker tasks call module application boundaries, preserve their registered names/arguments/schedules and retry behavior, and retain outbox atomicity and PostgreSQL `skip_locked` processing semantics.

### Tests for User Story 3

- [ ] T020 [P] [US3] Add unit tests for ARQ adapter delegation, registered task names, argument forwarding, and failure propagation in `tests/unit/workers/test_arq_worker_adapters.py`.

### Implementation for User Story 3

- [ ] T021 [US3] Update worker functions in `app/workers/arq_worker.py` to delegate subscription billing, abandoned-cart, and recommendation behavior to module application use cases while preserving task names, argument order/types, registrations, cron schedules, and process separation.
- [ ] T022 [US3] Move outbox processing and cleanup business behavior behind the owning module application/infrastructure boundaries in `app/modules/` and keep `app/workers/arq_worker.py` as the ARQ adapter; preserve atomic writes, `skip_locked`, attempt limits, backoff, terminal failures, and retention.
- [ ] T023 [US3] Validate worker success, retry, and terminal-failure behavior with `tests/test_outbox_worker.py`, `tests/test_subscription_billing.py`, and the worker adapter tests in `tests/unit/workers/test_arq_worker_adapters.py`.

**Checkpoint**: Background tasks retain their runtime interfaces and use the same application behavior as request-driven workflows.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Verify architecture boundaries and run the complete compatibility and quality gates.

- [ ] T024 [P] Review module imports and ownership across `app/modules/` to remove cross-module repository/ORM imports, domain imports of FastAPI/Pydantic transport schemas/SQLAlchemy sessions/infrastructure, duplicate implementations, and circular dependencies; review at least one use case from every migrated business module and record the selected use cases and results in `specs/001-layered-architecture-refactor/quickstart.md`.
- [ ] T025 Run the focused PostgreSQL-backed regression suite listed in `specs/001-layered-architecture-refactor/quickstart.md` and record results in `specs/001-layered-architecture-refactor/quickstart.md`.
- [ ] T026 Run `poetry run ruff check app/`, `poetry run mypy app/`, `poetry run pytest`, and the dependency audit from the existing CI security job; record results and resolve regressions in the responsible `app/modules/` files.
- [ ] T027 Verify a fresh-database `alembic upgrade head` and the existing Docker build/non-root checks defined in `.github/workflows/ci.yml`; record any environment limitation rather than treating an unavailable gate as a pass.
- [ ] T028 Document changed worker registrations, schedules, retry/idempotency behavior, terminal failure handling, and evidence-based operator recovery procedures in `specs/001-layered-architecture-refactor/quickstart.md`.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: T001 and T002 establish the API and ownership/runtime baselines; they can be performed independently.
- **Foundational (Phase 2)**: T003 depends on the ownership/contract inventory in T002 and blocks capability implementation.
- **User Story 1 (Phase 3)**: Depends on T003. Its module slices should follow the dependency map in `research.md`: account/catalog before cart/pricing, then order/payment/returns, then subscriptions/stored value and engagement. The new unit-test tasks are separate from implementation tasks per FR-007.
- **User Story 2 (Phase 4)**: Depends on User Story 1 because router, dependency, and model-discovery wiring targets the module-owned implementations. T019 runs after T014-T018.
- **User Story 3 (Phase 5)**: Depends on User Story 1 so worker adapters can call completed module application boundaries. T023 runs after T020-T022.
- **Polish (Phase 6)**: Depends on all selected user stories being complete. T028 also depends on the worker inventory and recovery procedure captured by T002 and the worker changes in T021-T022.

### User Story Dependencies

- **US1 (P1)**: Can start after Phase 2; delivers the MVP module ownership and inward dependency structure.
- **US2 (P1)**: Depends on US1; validates and completes compatible application/API/database wiring.
- **US3 (P2)**: Depends on US1; can proceed in parallel with US2 after the module application boundaries it uses are complete.

### Parallel Opportunities

- T001 and T002 can run in parallel because they capture separate baseline artifacts.
- Within US1, test tasks T004-T008 target distinct test files and can be authored in parallel; coordinate test interfaces with the ownership map.
- US1 capability slices may run in parallel only when the ownership map confirms they do not edit the same module or cross-module contract files.
- T020 can be developed independently of US2 wiring once the module application interfaces and worker task inventory are known.
- US2 and US3 implementation can proceed in parallel after US1, except for any shared edits to `app/dependencies.py`, `app/main.py`, or `app/workers/arq_worker.py`.

### Parallel Example: User Story 1

```text
After T002/T003 establish ownership and contracts:
Task: T004 - Add catalog/account use-case unit tests in tests/unit/modules/test_catalog_accounts.py
Task: T005 - Add cart/pricing use-case unit tests in tests/unit/modules/test_cart_pricing.py
Task: T007 - Add subscription/loyalty unit tests in tests/unit/modules/test_subscriptions_loyalty.py

Keep capability implementation tasks separate from these test-file changes, and serialize slices that share a contract or module.
```

## Implementation Strategy

### MVP First (User Story 1)

1. Complete setup and foundational tasks to establish OpenAPI, ownership, and shared-contract baselines.
2. Complete US1 module ownership slices, following dependency order in `research.md`.
3. Run each slice's new fake-port unit tests and representative existing tests before proceeding.
4. Stop and validate US1's independent architecture criteria before wiring the full application.

### Incremental Delivery

1. Deliver US1 module boundaries and use cases as the structural MVP.
2. Deliver US2 API/dependency/model-discovery wiring and verify public/admin compatibility.
3. Deliver US3 worker adapters and verify task/outbox behavior.
4. Finish with the full PostgreSQL-backed suite, Ruff, mypy, dependency audit, fresh-database migration, and Docker CI checks, and record worker operating and recovery procedures.
