# Data Model and Ownership

## Persistence impact

This feature changes code ownership, not the database schema. Existing SQLAlchemy models, constraints, enum values, foreign keys, Alembic history, and persisted records remain unchanged. No migration or backfill is planned.

## Domain entities

The following entities are already represented by ORM model modules under `app/models/`. This inventory describes domain groupings and key relations relevant to refactoring; it is not a schema redesign.

| Domain | Entities and key data | Relationships / ownership |
|---|---|---|
| Identity and account | `User`, `Address`, `OAuthAccount`, `AuditLog` | A user owns addresses and account-linked commerce records. Authentication/authorization remains at the API boundary, with account rules in services. |
| Catalog | `Product`, `ProductVariant`, `ProductImage`, `Category`, `Brand`, `ProductRelation`, `PriceHistory` | Products relate to catalog classifications, images, variants, and related products. CRUD owns lookups and persistence; services own catalog use cases and pricing decisions. |
| Cart and checkout | `Cart`, `CartItem`, `Coupon`, `CouponUsage`, `Promotion`, `Currency`, `TaxRate`, `Shipping` | A cart may be linked to a user or guest session and contains items; it can also carry currency, coupon, points, and store-credit selections. Checkout uses these records to create an order. |
| Orders and fulfillment | `Order`, `OrderItem`, `OrderEvent`, `Shipment`, `ReturnRequest` | An order is linked to its owner or guest email, address references/snapshots, currency, items, payment records, events, and shipment/return activity. Order status includes pending, paid, processing, packed, shipped, delivered, cancelled, payment-failed, refund, and return states. |
| Payments and stored methods | `Payment`, `PaymentEvent`, `SavedPaymentMethod` | A payment belongs to an order and records method, amount/currency, provider identifiers, status, and refund details. Payment/provider processing remains idempotent and must preserve webhook validation and duplicate-event behavior. |
| Inventory | `InventoryReservation`, `InventoryTransaction` | Reservations connect product/optional variant, quantity, expiry, optional user, and optional order. Reservation release must remain order-scoped; shared stock updates retain PostgreSQL row locking and transaction semantics. |
| Subscriptions | `Subscription` | A subscription links user, product/optional variant, saved payment method, shipping/billing addresses, interval, status, and next billing date. Renewal and retry/cancellation behavior remain unchanged. |
| Loyalty and stored value | `LoyaltyTransaction`, `GiftCard`, `StoreCreditTransaction` | These records represent points and stored-value balances/transactions. Checkout/refund behavior must retain base-currency accounting and reversible order snapshots. |
| Customer engagement | `Review`, `ReviewVote`, `Wishlist`, `ProductQuestion`, `ProductAnswer`, `BackInStockSubscription`, `PriceDropSubscription` | These records attach customer actions or subscriptions to users/products and feed notification workflows. |
| Side effects and recommendations | `OutboxEvent`, `ProductRelation` | `OutboxEvent` stores topic, payload, pending/completed/failed status, attempts, retry time, and processing timestamps. It is written atomically with the triggering business change; workers preserve locking, retry, and cleanup behavior. |

## Core field and state invariants

- `Order` retains its existing user-or-guest attribution, saved address references/snapshots, order number, base-currency monetary totals, customer currency/exchange-rate snapshot, discount/tax/shipping amounts, points/store-credit snapshots, order status, and payment status.
- `Payment.amount` and `refund_amount` are denominated in the payment's recorded currency, while order totals remain in the configured base currency.
- `Cart` retains user/session ownership, cart items, optional coupon/currency, points/store-credit selections, and activity timestamps.
- `InventoryReservation` remains tied to its exact order where present; never release another order's stock by grouping only on user or nullable guest owner.
- `Subscription.status` remains active/paused/past_due/cancelled, with existing billing date and retry state.
- `OutboxEvent.status` remains pending/processing/completed/failed; retry counts, backoff schedule, terminal failure, and retention cleanup remain as currently implemented.
- Existing model fields, defaults, constraints, relationships, and enums are authoritative; this refactor does not alter them.

## Target module ownership

These are capability groupings, not a mandatory one-module-per-entity rule. Final module ownership is established by the dependency inventory; do not create modules solely for symmetry.

| Candidate module | Owned concepts | Typical cross-module collaboration |
|---|---|---|
| `users` | User identity, account lifecycle, OAuth/MFA, user-owned settings | Other modules consume an application-level user identity/authorization contract rather than importing user persistence. |
| `addresses` | Address records and ownership validation | Orders and subscriptions request an address through a stable application boundary; they do not query address repositories directly. |
| `categories` | Category records and category use cases | Product/catalog use cases refer to category identifiers or contracts, not category ORM models. |
| `products` | Product, variant, image, inventory-facing catalog data, product relations/history | Cart/order/subscription workflows request product/variant data through an application port; inventory updates retain their atomic persistence boundary. |
| `brands` | Brand data and brand operations | Product/catalog workflows use stable identifiers or application contracts. |
| `carts` | Cart and cart-item lifecycle and selection state | Checkout invokes a cart application use case/port to obtain a validated checkout snapshot. |
| `orders` | Order lifecycle, order items/events, fulfillment/return workflows where cohesive | Payments, inventory, and notifications coordinate through application contracts and same-transaction boundaries where required. |
| `payments` | Payment/provider events, payment methods, refund workflows | Orders/subscriptions request payment operations through application-level contracts; provider adapters remain infrastructure. |
| `subscriptions` | Subscription state, billing schedule, retry state | Renewal uses products, saved payment, and orders through stable application contracts. |
| `pricing` or cohesive submodules | Currency, tax, shipping, coupons/promotions, points/stored-value calculations, as boundaries warrant | Checkout requests calculations/eligibility through application contracts; monetary invariants remain explicit. |
| `loyalty` / `gift_cards` | Loyalty and stored-value transactions and balances | Checkout/refund flows use application-level operations and retain reversible order snapshots. |
| `reviews`, `wishlists`, engagement modules | Reviews/votes, wishlists, Q&A, back-in-stock and price-drop subscriptions | Product/customer data is consumed through stable identifiers and application contracts. |
| `shared` (optional) | Small stable business contracts genuinely shared by multiple modules | Must not contain feature-specific persistence or unrelated utilities. |

## Code responsibilities inside modules

- `api/`: FastAPI routers, HTTP validation/authentication context, status mapping, and route dependency providers.
- `application/`: use cases, orchestration, transaction coordination, ports, and internal DTOs. Keep these framework-independent.
- `domain/`: entities, value objects, invariants, and domain errors where business complexity justifies the layer.
- `infrastructure/`: SQLAlchemy models/repositories and external-system implementations of module ports.
- `schemas.py`: Pydantic request/response shapes owned by the module; do not use transport schemas as domain entities.
- Small capabilities may combine application/domain code into a simple service and use fewer directories. Do not create empty packages just to make every module look alike.

Technical concerns such as configuration, logging, security utilities, database/session construction, observability, and external clients belong in focused `app/core/` or `app/infrastructure/` packages, not a catch-all shared business module.
