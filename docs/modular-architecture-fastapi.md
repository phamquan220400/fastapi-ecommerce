# Modular Layered Architecture for FastAPI

This design organizes the application by **business module** first and by **layer** inside each module. It keeps HTTP, business rules, and persistence separate while allowing each feature to evolve independently.

## Design goals

- Group code by business capability, such as `users`, `orders`, or `billing`.
- Keep FastAPI routes focused on HTTP concerns.
- Keep business rules independent of FastAPI and the database framework.
- Make dependencies point inward: adapters depend on application and domain code, never the reverse.
- Share infrastructure through explicit interfaces and dependencies, not direct access to another module's internals.

## Suggested project structure

```text
app/
├── main.py                         # Create the FastAPI app and register routers
├── api/
│   ├── router.py                   # Aggregate module routers under /api/v1
│   └── deps.py                     # Shared HTTP-level dependencies, if needed
├── core/
│   ├── config.py                   # Settings and environment loading
│   ├── errors.py                   # Shared application errors
│   ├── logging.py
│   └── security.py                 # Token/password utilities; no route handlers
├── infrastructure/
│   ├── database.py                 # Engine/session creation
│   ├── unit_of_work.py             # Transaction boundary, if used
│   └── observability.py            # Tracing/metrics setup
├── modules/
│   ├── users/
│   │   ├── api/
│   │   │   ├── router.py           # Endpoints and HTTP status mapping
│   │   │   └── dependencies.py     # Resolve user use cases for routes
│   │   ├── application/
│   │   │   ├── use_cases/
│   │   │   │   ├── create_user.py
│   │   │   │   └── get_user.py
│   │   │   ├── ports.py            # Repository/service interfaces
│   │   │   └── dto.py              # Internal application inputs/outputs
│   │   ├── domain/
│   │   │   ├── entities.py         # User entity and invariants
│   │   │   ├── value_objects.py
│   │   │   └── errors.py
│   │   ├── infrastructure/
│   │   │   ├── models.py           # SQLAlchemy table models
│   │   │   └── repositories.py     # SQLAlchemy implementation of ports
│   │   └── schemas.py              # Pydantic request/response schemas
│   └── orders/
│       └── ...                     # Same structure when the module needs it
└── tests/
    ├── unit/
    ├── integration/
    └── api/
```

Start with fewer folders if a module is small. Add a separate `domain/` or `application/` package when the feature has enough business logic to justify the distinction; avoid empty layers created only for symmetry.

## Layer responsibilities

| Layer | Owns | Should not own |
| --- | --- | --- |
| `api` | Routes, HTTP validation, authentication context, status codes | Business rules, SQL queries, transaction details |
| `application` | Use cases, orchestration, authorization decisions, transaction coordination | FastAPI request/response objects, framework-specific ORM queries |
| `domain` | Business entities, value objects, invariants, domain errors | FastAPI, Pydantic transport schemas, SQLAlchemy sessions |
| `infrastructure` | Database models, repository implementations, external clients | Endpoint behavior or business policy |
| `schemas.py` | Pydantic request and response contracts for the module | Database access or business workflows |
| `core` | Cross-cutting configuration and technical utilities | Feature-specific business behavior |

For a simple CRUD service, `application` and `domain` can begin as a single `service.py` and grow into packages when the rules become substantial. Keep the dependency direction the same.

## Dependency direction

```text
FastAPI router → application use case → domain
                         ↑
             infrastructure adapter
```

- A router calls a use case; it does not query the database directly.
- A use case depends on repository protocols declared in `application/ports.py`.
- A repository implementation in `infrastructure/` fulfills that protocol.
- FastAPI dependency providers wire the use case to its infrastructure dependencies.
- The domain imports no application, API, or infrastructure modules.
- One business module should not reach into another module's repository or ORM model. Expose a use case or application-level port for cross-module work.

## Example: create-user flow

```python
# modules/users/application/ports.py
from typing import Protocol

from app.modules.users.domain.entities import User


class UserRepository(Protocol):
    async def add(self, user: User) -> User: ...
    async def get_by_email(self, email: str) -> User | None: ...
```

```python
# modules/users/application/use_cases/create_user.py
from app.modules.users.application.ports import UserRepository
from app.modules.users.domain.entities import User


class CreateUser:
    def __init__(self, users: UserRepository) -> None:
        self._users = users

    async def execute(self, email: str, display_name: str) -> User:
        existing = await self._users.get_by_email(email)
        if existing is not None:
            raise ValueError("A user with this email already exists")

        user = User.create(email=email, display_name=display_name)
        return await self._users.add(user)
```

```python
# modules/users/api/router.py
from fastapi import APIRouter, Depends, HTTPException, status

from app.modules.users.api.dependencies import get_create_user
from app.modules.users.application.use_cases.create_user import CreateUser
from app.modules.users.schemas import UserCreate, UserResponse

router = APIRouter(prefix="/users", tags=["users"])


@router.post("", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(
    body: UserCreate,
    use_case: CreateUser = Depends(get_create_user),
) -> UserResponse:
    try:
        user = await use_case.execute(
            email=body.email,
            display_name=body.display_name,
        )
    except ValueError as exc:
        raise HTTPException(status_code=409, detail=str(exc)) from exc

    return UserResponse.model_validate(user)
```

The example keeps the endpoint thin. In a larger application, replace generic exceptions such as `ValueError` with a domain/application error and map that error to HTTP in the API layer.

## Dependency wiring

Create concrete infrastructure objects at the application boundary, then inject them into use cases. For example, `modules/users/api/dependencies.py` can provide `CreateUser` using the request-scoped database session and `SqlAlchemyUserRepository`. This keeps the router unaware of SQLAlchemy and lets unit tests pass a fake repository directly.

Use one session per request or an explicit unit-of-work boundary. Define who commits and rolls back: commonly the use case or unit of work owns the transaction, while repository methods only issue queries and writes.

## Module boundaries

- Keep each module's routes, schemas, use cases, domain, and persistence code together.
- Share a capability through a stable application service or port, rather than importing another module's tables.
- Put truly cross-cutting technical code in `core/` or `infrastructure/`; keep business concepts in their owning module.
- Avoid a global `models.py`, `schemas.py`, or `services.py` that accumulates every feature.
- If two modules need the same domain concept, decide which module owns it or define a small shared contract. Avoid a broad `common` package for unrelated code.

## Testing boundaries

- **Unit:** test domain rules and application use cases with fake ports/repositories.
- **Integration:** test repository implementations against a disposable or test database.
- **API:** test routes with FastAPI's test client and dependency overrides.
- Keep business-rule tests independent of the web server and database where possible.

## Practical starting point

For a small service, begin with `main.py`, `core/config.py`, and one folder per module containing `router.py`, `schemas.py`, `service.py`, and `repository.py`. Split `service.py` into application use cases and domain objects as business rules grow. The goal is clear ownership and dependency direction, not the largest possible directory tree.
