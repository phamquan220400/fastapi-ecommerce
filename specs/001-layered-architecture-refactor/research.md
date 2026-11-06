# Research: Modular Layered Architecture Refactor

## Architecture reference

`docs/modular-architecture-fastapi.md` is the target architecture. It groups code by business capability first, then by layer within a module. It explicitly allows small modules to use fewer folders and requires inward dependencies, application-level cross-module contracts, and focused shared technical packages.

## Findings and decisions

### Organize by business capability, not global layer

- **Decision**: Move feature code from global route/service/CRUD/model/schema folders into `app/modules/<capability>/`, with module-owned API, application, optional domain, schemas, and infrastructure.
- **Rationale**: The repository currently separates files by layer across the whole application. The requested reference instead colocates a capability and its responsibilities. This makes ownership visible and limits unrelated modules' implementation details.
- **Alternatives considered**: Keep the current global layer folders and only enforce dependencies between them; rejected because it does not meet the module-first requirement.

### Add only layers justified by module complexity

- **Decision**: Use `api/`, `application/`, `domain/`, and `infrastructure/` where meaningful. A simple module may begin with a small router, schema, service/use case, and repository rather than empty package tiers.
- **Rationale**: The reference explicitly warns against empty layers and unnecessary abstractions. Existing capabilities vary substantially in complexity; checkout, payments, and subscriptions need more explicit application/domain boundaries than simple catalog CRUD.
- **Alternatives considered**: Generate an identical deep folder tree for every capability; rejected as ceremony without value.

### Keep domain code independent and use ports for collaboration

- **Decision**: Domain code must not import FastAPI, Pydantic transport schemas, SQLAlchemy sessions, or infrastructure. Application use cases own orchestration and depend on declared ports; infrastructure implements persistence/external adapters. Cross-module collaboration uses another module's stable use case or an application port rather than its repository or ORM model.
- **Rationale**: This follows the attached dependency rule and enables business logic to be tested independently. It prevents imports across module internals from recreating the coupling in a new directory layout.
- **Alternatives considered**: Direct cross-module ORM/repository imports or a global service/repository; rejected because both leak implementation and create circular-dependency risk.

### Keep shared code narrow

- **Decision**: Retain focused technical configuration, security, logging, errors, database/session management, and observability in `app/core/` and `app/infrastructure/`. Create `app/modules/shared/` only for small stable business contracts genuinely owned across modules.
- **Rationale**: The user requested shared and category-style modules, while the reference cautions against broad common packages. This separates technical reuse from shared domain language and avoids a new catch-all.
- **Alternatives considered**: Put all utilities and shared models into a large `shared` module; rejected because it would become a global dependency and obscure ownership.

### Preserve API and persistence contracts while moving files

- **Decision**: Keep the existing public/admin OpenAPI contract, PostgreSQL schema, migration chain, and business behavior unchanged. Recompose routes through an API router and update application wiring; preserve SQLAlchemy metadata/model discovery for Alembic and tests.
- **Rationale**: The refactor changes internal ownership only. Current setup imports models centrally for metadata discovery, so moving model modules must explicitly preserve this behavior and avoid duplicate model definitions.
- **Alternatives considered**: Combine the reorganization with API or schema redesign, database normalization, or dependency replacement; rejected as outside scope and too risky for a structural migration.

### Keep ARQ as the worker adapter

- **Decision**: Preserve the separate ARQ process, task names/arguments, registrations, cron schedules, retries, and operational behavior. Worker functions call module application behavior rather than duplicating business rules.
- **Rationale**: The existing worker module is a runtime interface, and the outbox processor relies on PostgreSQL `skip_locked`, retry/backoff, and transaction ordering.
- **Alternatives considered**: Replace ARQ or introduce a broker; rejected because infrastructure redesign is out of scope.

### Validate database behavior with PostgreSQL

- **Decision**: Continue using the existing pytest/Testcontainers PostgreSQL environment for integration and API behavior. Add unit tests around domain/use-case ports in separate implementation tasks as required by FR-007.
- **Rationale**: The project depends on PostgreSQL row locking and transaction semantics, which mocks or SQLite do not establish. The architecture reference recommends fake ports for unit tests and a real disposable database for repository integration.
- **Alternatives considered**: Replace existing PostgreSQL integration tests with unit tests; rejected because both test levels are needed for their distinct guarantees.

## Candidate module ownership

The initial module map follows current feature files and remains subject to dependency inventory. Candidate modules include `users`, `addresses`, `categories`, `products`, `brands`, `carts`, `orders`, `payments`, `subscriptions`, `pricing` (including currency/tax/shipping only if cohesive), `loyalty`, `gift_cards`, `returns`, `promotions`, `coupons`, `reviews`, `wishlists`, and search/notifications where they represent separable capabilities. Do not split or combine modules merely to match this list; assign a single owner and use stable contracts for cross-module workflows.

## Repository observations

- The current layer directories include 25 route modules, 23 services, 21 CRUD modules, 42 model modules, and 28 schema modules; these are a migration inventory, not a required one-to-one module count.
- `app/dependencies.py` is the current service composition point and `app/api/v1/init_routes.py` aggregates routers. The new composition root/router must replace these responsibilities coherently.
- Direct ORM/database code exists in some routes, and use-case decisions/orchestration exist in some CRUD modules; these should move to their owning module layers as each capability is migrated.
- `app/models/__init__.py` registers model imports for Alembic and test metadata creation. Its responsibility must be replaced or retained as a compatibility/discovery mechanism until all model registration paths are updated.
- The worker remains at `app/workers/arq_worker.py` and uses synchronous SQLAlchemy sessions with async ARQ tasks.
- No new dependency or infrastructure component is needed to implement this package reorganization.

## Unresolved questions

No architecture choice blocks planning. Exact module grouping and migration order should be finalized in implementation tasks after mapping cross-module calls and data ownership, with no module reaching into another module's persistence internals.
