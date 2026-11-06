# API and Worker Compatibility Contract

## Scope

This is an internal refactor. Existing customer-facing and administrative HTTP operations and ARQ task interfaces remain stable; no endpoint, request/response schema, status code, authorization rule, job name, task argument, or schedule is intentionally added, removed, or changed.

## HTTP API contract

- The registered route set, methods, paths, and prefixes remain unchanged. Internal aggregation may move from `app/api/v1/init_routes.py` to `app/api/router.py`.
- Existing request and response models in `app/schema/` remain the public serialization contract.
- Preserve methods, paths, response status codes, response body shapes, validation errors, authorization/ownership checks, rate limits, and documented OpenAPI metadata.
- Preserve route ordering where static paths coexist with path-parameter routes.
- Before and after each refactor slice, compare the generated OpenAPI document (paths, methods, parameters, request/response schemas, security requirements, and response codes). Unexpected differences block the slice.
- Preserve idempotency-key behavior and replay shape for order/payment creation, including user/path/key isolation.
- Module API adapters call application-level use cases; route aggregation and dependency wiring may change internally without changing the HTTP contract.

## Background-job contract

- Preserve the ARQ task names, argument order/types, result behavior, function registrations, and cron schedules in `app/workers/arq_worker.py`.
- Preserve outbox polling semantics: eligible pending events only, PostgreSQL `skip_locked`, attempt limit, exponential retry delay, terminal failure, post-commit email dispatch, and completed-event retention.
- Preserve subscription billing, abandoned-cart, and recommendation task outcomes and failure/retry visibility.
- Keep the API process and ARQ worker as separate runtime processes.
- Worker adapters invoke module application behavior and do not import another module's repositories or ORM models.

## Database and side-effect contract

- No model, migration, index, enum, constraint, or persisted field changes are part of this feature.
- Preserve each workflow's transaction boundaries, row locks, idempotency, and state transitions.
- Business state and required outbox records must continue to commit atomically.
- External integrations and optional services keep their existing failure behavior; no new infrastructure dependency is introduced.
- A module does not import another module's persistence implementation. Cross-module workflows use stable application-level use cases or ports; domain code remains independent of FastAPI, Pydantic transport schemas, SQLAlchemy sessions, and infrastructure.
- SQLAlchemy metadata discovery for Alembic and test schema setup remains complete after model files move into module infrastructure; only one authoritative mapped class per persisted table is registered.

## Acceptance evidence

Use the existing domain tests plus the compatibility checks in `quickstart.md`. Any intentional contract change must be split into a separately specified and approved change, not hidden in the refactor.
