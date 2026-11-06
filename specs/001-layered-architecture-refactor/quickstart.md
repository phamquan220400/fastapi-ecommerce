# Quickstart: Refactor Compatibility Validation

This guide validates that the module-first reorganization preserves representative commerce, API, database, and worker behavior. It assumes the implementation has been completed in one or more capability slices.

## Prerequisites

- Python 3.11 or newer and Poetry installed.
- Docker running for the PostgreSQL Testcontainers suite.
- Repository dependencies installed:

  ```bash
  poetry install --with dev
  ```

No `DATABASE_URL` is required for pytest; the session fixture starts PostgreSQL and each test receives isolated data.

## Validate representative contracts and workflows

1. Record the current OpenAPI document and compare it with the post-refactor app's generated OpenAPI. Confirm paths, methods, schemas, statuses, and security metadata are unchanged even if router aggregation moved. See [API compatibility contract](./contracts/api-compatibility.md).
2. Run representative customer and operational flows:

   ```bash
   poetry run pytest \
     tests/test_orders.py \
     tests/test_guest_checkout.py \
     tests/test_checkout_pricing.py \
     tests/test_idempotency.py \
     tests/test_concurrency_locking.py \
     tests/test_outbox_worker.py \
     tests/test_subscription_billing.py \
     tests/test_returns.py \
     tests/test_refunds.py
   ```

   Expected: all tests pass against PostgreSQL; successful flows preserve their resulting state and failure flows preserve their existing HTTP/error outcomes without partial state changes.

3. Run static checks:

   ```bash
   poetry run ruff check app/
   poetry run mypy app/
   ```

   Expected: both complete without new diagnostics.

4. Run the complete behavior suite:

   ```bash
   poetry run pytest
   ```

   Expected: the full PostgreSQL-backed suite passes.

5. Run the dependency audit from the existing CI security job. Use the Poetry dependency export and `pip-audit` flow defined in `.github/workflows/ci.yml`; do not hand-edit the generated `requirements.txt`. Expected: the audit completes and any findings are recorded and resolved or explicitly reported.

6. In CI or a configured PostgreSQL environment, run the fresh-database migration and Docker checks already defined by the project. Expected: Alembic upgrades an empty database to head and the image builds/runs using its existing non-root configuration.

## Architecture acceptance checks

- Trace a representative workflow through its capability module: API/worker adapter → application use case → port → infrastructure implementation. Include domain code only where it provides meaningful invariants.
- Review at least one use case from every migrated business module. Record each module, selected use case, whether it enters through the application boundary, and any route/neighbor access to persistence internals.
- Trace and record at least these five workflows end to end: catalog, account, checkout, subscription billing, and outbox processing. For each, identify its API/worker adapter, application behavior, and infrastructure implementation, and note any circular dependency or cross-module ORM/repository import.
- Confirm each capability's related API, application, schemas, domain, and persistence code is colocated under `app/modules/<capability>/` where those layers exist.
- Confirm routes do not execute SQL or own commerce decisions; domain code imports no FastAPI, Pydantic transport schema, SQLAlchemy session, or infrastructure modules.
- Confirm modules do not import another module's repositories or ORM models. Cross-module collaboration uses application-level use cases or ports.
- Confirm small modules have no empty layers, `shared` contains only stable cross-module contracts, and no duplicate authoritative implementation or circular dependency remains.
- Confirm all SQLAlchemy models are registered for Alembic and test metadata creation without registering duplicate mappings.
- Confirm worker function registration, cron schedules, transaction/outbox coupling, retries, and failure visibility remain intact.

These are architecture review checks, not substitutes for the PostgreSQL behavior tests above.

## Worker operations and recovery

For each changed asynchronous worker, record its registered task name and enqueue/cron path, arguments, retry and idempotency behavior, terminal failure handling, and the existing operator recovery procedure. Derive recovery steps from the worker implementation and current operational conventions; do not assume failed jobs are automatically replayed or that an outbox record proves external delivery. Include the relevant log/metric/trace signal and any manual action or limitation needed to recover safely.
