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

## Phase 1 ownership and dependency inventory

### Initial ownership map

The map below assigns one target owner to each existing capability. Keep small capabilities simple, and route cross-module collaboration through application-level use cases or ports instead of carrying the current persistence imports into the new modules.

| Target owner | Existing code and responsibility | Dependencies to replace with application contracts |
|---|---|---|
| `users` | `user` and `oauth_account` models/CRUD, `user_service.py`, user routes | Addresses and email delivery are separate capabilities; authentication remains an API boundary concern. |
| `addresses` | `address` model/CRUD/service; address operations currently share `user.py` routes | Orders and subscriptions need validated address snapshots/identifiers, not address persistence. |
| `categories`, `brands`, `products` | Corresponding models, CRUD, services, and route modules; product variants, images, relations, and history belong with products | Products refer to categories/brands by stable IDs; cart/order/subscription workflows request product data through application contracts. |
| `carts` | Cart/cart-item models and CRUD, cart service/routes | Product/variant validation and pricing/currency/coupon calculations cross capability boundaries. |
| `pricing` | `services/pricing.py` plus currency, tax, shipping, coupon, and promotion calculations/configuration | Checkout requests price, tax, shipping, discount, and currency operations through stable application contracts. Coupon/promotion records may remain separate modules if their use cases justify it. |
| `orders`, `inventory`, `returns` | Order/item/event/shipment models and order service/CRUD; reservation and inventory transaction models; return routes currently split between order/admin | Orders coordinate stock, payment, addresses, pricing, loyalty, stored value, and outbox writes; these require ports and a shared transaction boundary, not imported peer repositories/models. |
| `payments` | Payment/event/saved-method models, payment and saved-method services/CRUD/routes; refund paths in payment/admin | Payment operations consume order and subscription contracts; provider calls stay behind payment infrastructure. |
| `subscriptions` | Subscription model/service/CRUD/routes and scheduled renewal behavior | Renewal requires product, address, saved-payment, order, and payment contracts. |
| `loyalty`, `gift_cards` | Loyalty, gift-card, and store-credit models/CRUD/routes and their checkout/refund rules | Checkout, cancellation, and refund workflows consume application contracts and preserve reversible order snapshots/base-currency accounting. |
| `coupons`, `promotions` | Coupon/promotion models, CRUD, services, and routes | Cart/checkout request eligibility and application of discounts through pricing contracts. |
| `reviews`, `wishlists`, customer engagement | Review/vote, wishlist, product Q&A, back-in-stock, and price-drop models/services/routes | Product and user data use identifiers/contracts; notification delivery and outbox writes remain separate adapters. |
| `search` | Elasticsearch service and `/elastic` routes | Elasticsearch remains optional; catalog use cases do not depend on its availability. |
| Technical composition | `app/core/`, `app/db/`, middleware, `app/dependencies.py`, `app/api/v1/init_routes.py`, and `app/workers/` | Keep technical dependencies in focused infrastructure/core packages; composition roots wire module boundaries without owning business rules. |

### Current boundary violations and migration dependencies

- Most routes obtain a service via `app/dependencies.py`, but direct persistence or model access remains in `admin.py`, `audit.py`, `currency.py`, `gift_card.py`, `healthcheck.py`, `loyalty.py`, `order.py`, `shipping.py`, `tax_rate.py`, and `user.py`. `admin.py`, `audit.py`, `shipping.py`, and `tax_rate.py` execute ORM queries in the route; `currency.py`, `gift_card.py`, and `loyalty.py` call CRUD directly; `order.py` queries `ReturnRequest` and constructs `OrderCrud`; `healthcheck.py` calls the database health check. `user.py` imports the `User` model and address service. These are explicit route-migration targets, not patterns to preserve.
- `OrderCrud` is the broadest orchestration seam: it imports address/currency CRUD and pricing, plus product/cart/coupon, inventory, loyalty/store-credit, subscription, shipment, user, and outbox models. Its order construction, stock reservation, price calculation, stored-value effects, and outbox write currently share a SQLAlchemy session and commit.
- `PaymentService` combines payment/order/saved-method CRUD with order, inventory, loyalty, store-credit, and user models. `SubscriptionCrud` constructs `OrderCrud`; `SubscriptionService` uses saved-method CRUD and product models; `SavedPaymentMethodService` calls subscription CRUD. Move these collaborations behind use cases/ports while preserving transaction coordination.
- `CartService` combines cart/product CRUD with pricing, currency CRUD, and direct product-variant, coupon, tax, and address models. `WishlistService` calls product and cart-item CRUD. `PromotionService` reads category/product models. These are cross-capability dependencies that should become application contracts.
- The product persistence layer imports back-in-stock and price-drop services, while those services also depend on persistence, creating an undesirable persistence-to-service edge. `AdminService` likewise combines user/order/review/product models and CRUD. Keep these out of the new module infrastructure direction.
- `app/dependencies.py` currently wires services directly; `app/api/v1/init_routes.py` registers 25 routers in significant order (including guest order routes within their router and prefix-owning tax/shipping routers). `app/main.py` calls `init_routes(app)`. Preserve route ordering/prefix behavior while replacing these composition responsibilities.

### Persistence, transaction, and discovery invariants

- `OrderCrud.validate_stock` locks each selected product or variant with `with_for_update()` before checking availability. `ProductCrud` has another row-locking stock path. Order placement must retain row-lock scope and decrement/reservation behavior under PostgreSQL.
- Order construction commits order state, order items/reservations, applied loyalty/store-credit effects, and the `order.created` outbox row using the same session/transaction. The outbox record is not proof of broker delivery: `_publish_event` currently logs because no message broker is configured.
- The outbox worker selects eligible pending events in creation order, batches at most 50, and uses `with_for_update(skip_locked=True)` so concurrent pollers do not block on or double-process locked rows. It commits event state/retry changes before sending back-in-stock and price-drop email side effects. A send failure after that commit does not automatically make the completed event eligible again.
- Outbox publish failures increment `retry_count`; attempts below five schedule exponential delays of 2, 4, 8, then 16 minutes (capped at 60 minutes), and the fifth failure sets terminal `failed`, stores `error_message`, and clears `next_attempt_at`. Completed rows older than 30 days are deleted. No built-in operator requeue/replay or dead-letter workflow was found; terminal rows and task logs require owner review and an explicitly controlled recovery decision.
- Order/payment creation routes use the existing idempotency dependency and cache successful response payloads. Preserve key isolation, replay shape, webhook duplicate handling, and business/database state transitions during movement.
- Subscription billing retries internal failures after 1, 3, and 7 days; `record_renewal_failure` marks the subscription `past_due` until the retry budget is exceeded, then cancels it. Abandoned-cart detection marks the cart before email delivery, so a send failure is not automatically retried unless later cart activity makes it eligible again.
- Expired inventory reservations are cleaned by a separate API-lifespan loop every five minutes, not by ARQ. ARQ is a separate process and uses synchronous SQLAlchemy sessions inside async tasks; keep PostgreSQL integration tests for locks and transaction semantics.
- `app/models/__init__.py` imports all mapped model modules, and `alembic/env.py` imports `app.models` before setting `target_metadata = Base.metadata`. Tests also rely on complete metadata registration. Replace or retain this discovery path so each mapped class is registered exactly once after files move.

### ARQ runtime and recovery inventory

`WorkerSettings.functions` registers nine tasks: `send_order_confirmation_email_task(to_address, order_number, total_amount)`, `send_password_reset_email_task(to_address, reset_token)`, `send_verification_email_task(to_address, verification_token)`, `send_guest_order_claim_email_task(to_address, order_number, claim_token)`, `detect_abandoned_carts_task()`, `process_outbox_events_task()`, `cleanup_completed_outbox_events_task()`, `compute_frequently_bought_together_task()`, and `process_due_subscriptions_task()`. The four email functions are enqueue targets and are not cron jobs.

| Scheduled task | Existing schedule | Retry/idempotency and terminal behavior |
|---|---|---|
| `detect_abandoned_carts_task` | Daily at 00:00 and 12:00 | Selects carts inactive for 24 hours; `abandoned_email_sent_at` prevents repeat reminders for the same abandonment, but is committed before sending. |
| `process_outbox_events_task` | Every 30 seconds (`second={0, 30}`) | PostgreSQL `skip_locked`, 50-event batches, five publish attempts, exponential backoff, terminal `failed` state, then no automatic requeue. Email side effects occur post-commit. |
| `cleanup_completed_outbox_events_task` | Daily at 03:00 | Deletes completed rows after 30 days; failed rows are not pruned by this task. |
| `compute_frequently_bought_together_task` | Daily at 01:00 | Reconciles only auto-generated relations from paid-order history; no separate retry/recovery path is implemented. |
| `process_due_subscriptions_task` | Daily at 05:00 | Retries per-subscription after 1/3/7 days and cancels after the retry budget; Stripe and order failures are recorded through subscription state. |

The worker also sets `max_jobs=10` and `job_timeout=300`; it does not override ARQ's task retry defaults. No supported operator procedure for manually replaying failed ARQ jobs or terminal outbox rows is documented in the repository. Recovery documentation must not imply automatic replay or successful external delivery; inspect persisted status/error and logs, reconcile the underlying business side effect, and require an explicit owner-approved replay procedure where one is needed.

## Unresolved questions

No architecture choice blocks planning. Exact module grouping and migration order should be finalized in implementation tasks after mapping cross-module calls and data ownership, with no module reaching into another module's persistence internals.
