<!--
Sync Impact Report
Version change: unratified scaffold -> 1.0.0
Modified principles: none; scaffold placeholders replaced.
Added sections: Core Principles; Engineering Constraints; Development Workflow; Governance.
Removed sections: none.
Deferred items: none.
-->

# FastAPI E-Commerce Constitution

## Core Principles

### I. Correctness of Commerce State

- Changes to orders, payments, refunds, subscriptions, inventory, discounts, loyalty points,
  or gift cards MUST state the affected invariants and test both success and failure paths.
- Shared inventory and counters MUST be updated atomically, using database constraints,
  transactions, and row locks where needed. Concurrency-sensitive changes MUST include a
  PostgreSQL test exercising competing operations.
- Money and loyalty calculations MUST preserve the existing base-currency accounting and
  conversion rules. Changes to rounding or conversion MUST include boundary-value tests.
- Payment- and order-creating requests MUST preserve idempotent replay and user isolation;
  webhook and worker changes MUST account for duplicate delivery and retries.
- Domain state changes and their required outbox records MUST commit in the same transaction.
  An outbox record MUST NOT be described as proof of external delivery when no publisher exists.

### II. Security at Every Boundary

- Each new or changed endpoint MUST declare its authentication and authorization requirements,
  validate input, and test permitted and denied access, including resource ownership where
  applicable. Public endpoints MUST be explicitly identified in the change.
- Credentials, tokens, payment details, and personal data MUST NOT be committed or emitted in
  logs, metrics, traces, or error responses. New external integrations MUST document their
  credential configuration and failure behavior without exposing secret values.
- Changes to payment webhooks MUST verify provider authenticity before processing and test
  invalid signatures and duplicate events. Abuse-sensitive endpoints MUST specify and test
  their rate-limiting behavior.
- Security fixes MUST include a regression test or, when automation cannot reproduce the issue,
  documented reproducible verification and its limitation in the pull request.

### III. Explicit Boundaries and Simple Design

- Routes MUST handle HTTP concerns and obtain services through dependency injection; services
  MUST own business rules; CRUD modules MUST own database queries. New model classes MUST be
  registered for Alembic discovery.
- A new abstraction, service, external system, or performance optimization MUST identify the
  concrete problem it solves and the simpler alternative considered in the pull request.
- Failures MUST be surfaced through the existing error or logging conventions; code MUST NOT
  silently convert errors into successful responses or conceal invalid input.
- Performance changes MUST include a reproducible baseline and comparison for the affected
  operation, and MUST preserve correctness and security tests.

### IV. Compatible, Migratable Contracts

- Public API changes MUST identify affected endpoints, request and response schemas, status
  codes, authentication requirements, and existing consumers. Additive changes MUST preserve
  existing behavior unless a breaking change is explicitly approved.
- A breaking API or event contract change MUST include an approved migration plan, a documented
  deprecation and removal schedule, a consumer communication plan, and compatibility tests for
  the supported transition. Emergency exceptions follow the governance exception process.
- Schema changes MUST use Alembic, preserve the existing migration chain, and include a plan
  for existing data and rollback or forward recovery. Migrations MUST work on both an existing
  database and a fresh database.

### V. Observable and Recoverable Operations

- Changes to requests, background jobs, or external integrations MUST expose failures through
  existing structured logging, metrics, or tracing as appropriate. Logs MUST identify the
  operation and outcome and include a correlation identifier when available.
- New or changed asynchronous jobs MUST document their registration, retry and idempotency
  behavior, terminal failure handling, and operator recovery procedure; tests MUST cover
  success and failure or retry paths.
- New infrastructure dependencies MUST document startup and outage behavior, configuration,
  and recovery steps. Optional services MUST NOT silently become mandatory for unrelated flows.

### VI. Verified Change and Sustainable Maintenance

- Every behavior change MUST include automated tests for the changed contract and relevant
  failure cases. Bug fixes MUST add a regression test, subject to the documented exception
  for non-automatable security verification in Principle II.
- PostgreSQL-specific behavior, transactions, locks, and migrations MUST be tested against
  PostgreSQL; SQLite or mocks alone MUST NOT be used as evidence for those behaviors.
- Refactoring MUST preserve externally observable behavior and existing tests. Any intentional
  behavior change MUST be identified, documented, and tested as a separate contract change.
- Documentation MUST be updated in the same pull request when setup, configuration, API
  behavior, operational procedures, or consumer migration steps change.

## Engineering Constraints

- The decision order for competing goals is **correctness, security, simplicity,
  maintainability, performance optimization, developer convenience**. A pull request MUST
  identify the conflict and explain the choice against this order; lower-ranked goals MUST
  NOT override higher-ranked ones. Security requirements remain mandatory.
- PostgreSQL, SQLAlchemy, Alembic, FastAPI, and the separate ARQ worker are the established
  runtime boundaries. A proposal that replaces or adds an infrastructure component MUST
  document operational ownership, deployment, migration, failure modes, and recovery.
- Use the project's naive-UTC time helper for database timestamps, preserve base-currency
  storage, and register worker tasks for the intended enqueue or scheduled execution path.
- Poetry is the dependency source of truth. A dependency addition MUST document its purpose,
  alternatives including existing dependencies, maintenance and license assessment, security
  audit result, and runtime or operational impact. Update the lockfile and regenerate the
  Docker requirements export when affected; do not hand-edit the export.

## Development Workflow

- Every pull request MUST describe its scope, affected invariants and contracts, test evidence,
  and operational or migration impact. It MUST link to updated documentation or state why
  none is needed.
- At least one reviewer other than the author MUST approve each pull request. Review MUST
  explicitly check applicable principles, security and authorization boundaries, compatibility,
  migrations, test coverage, and observability; unresolved blocking findings MUST be addressed
  before merge.
- Before merge, the project's CI gates MUST pass: Ruff, mypy, the PostgreSQL-backed pytest
  suite, dependency audit, fresh-database Alembic upgrade, and Docker build and non-root
  checks. A failed or unavailable gate is not a pass and requires a governance exception.
- Changes to an existing database schema MUST also provide evidence of migration against a
  representative existing database, including relevant data transformations.
- Tests MUST include representative error and boundary cases; concurrency tests MUST verify
  the resulting invariant rather than only asserting that requests completed.

## Governance

- This constitution governs future development and pull request review. When existing code
  does not comply, changes MUST NOT worsen the divergence; the pull request MUST identify
  affected pre-existing gaps and either address them within scope or link a follow-up issue.
- Amendments MUST be proposed in a pull request describing the motivation, affected principles,
  compatibility impact, and migration or adoption plan. At least one reviewer other than the
  author MUST approve the amendment before merge.
- Constitution versions follow semantic versioning: MAJOR for backward-incompatible removal
  or redefinition of governance, MINOR for new principles or materially expanded guidance,
  and PATCH for non-semantic clarifications. The amendment MUST update the version, amendment
  date, and sync impact report; the original ratification date MUST remain unchanged.
- An exception MUST be recorded in the pull request before merge, with the specific rule,
  reason, risk, compensating controls, owner, expiry date, and linked remediation issue.
  A reviewer other than the author MUST approve it. Exceptions MUST NOT be treated as a
  successful test or CI result; expired exceptions require renewed approval or remediation.
- Reviewers MUST check compliance on every pull request. Material unresolved correctness or
  security risks block merge; an exception cannot authorize knowingly unsafe behavior.

**Version**: 1.0.0 | **Ratified**: 2026-11-06 | **Last Amended**: 2026-11-06
