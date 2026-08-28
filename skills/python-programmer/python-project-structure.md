---
name: python-project-structure

description: >
  Mandatory FastAPI / backend project directory layout and structural rules.
  Every project must follow this layout regardless of size or scope.
  Create every directory even if empty to preserve consistency.
---

# Python Project Structure

## Directory Layout

The following directory layout is **mandatory for every FastAPI / backend project**, regardless of size or scope. If a project spec does not require a given layer, still create the corresponding directory (and a minimal `__init__.py` or placeholder module) so the structure remains consistent and future-proof.

```
myapp/
├── pyproject.toml
├── src/
│   └── myapp/
│       ├── __init__.py
│       ├── main.py                    # FastAPI app factory, lifespan, middleware
│       ├── config.py                  # Pydantic Settings (env validation)
│       │
│       ├── domain/                    # Pure logic, zero I/O
│       │   ├── __init__.py
│       │   ├── models/                # Pydantic I/O contracts + internal value objects
│       │   │   ├── __init__.py
│       │   │   └── *.py               # One file per domain entity (User, Order, etc.)
│       │   ├── enums/                 # Exhaustive enums for match-case
│       │   │   ├── __init__.py
│       │   │   └── *.py               # One file per enum (order_status.py, user_role.py)
│       │   ├── protocols/             # @runtime_checkable traits
│       │   │   ├── __init__.py
│       │   │   ├── repository.py      # Repository[T, ID] protocol
│       │   │   └── unit_of_work.py   # UnitOfWork protocol
│       │   ├── services/              # Pure functions, no classes
│       │   │   ├── __init__.py
│       │   │   └── *.py               # One file per service (user_service.py)
│       │   └── errors/                # Explicit error types
│       │       ├── __init__.py
│       │       ├── base.py            # DomainError base
│       │       └── *.py               # One file per error variant
│       │
│       ├── infrastructure/              # I/O boundaries, side effects only here
│       │   ├── __init__.py
│       │   ├── db/
│       │   │   ├── __init__.py
│       │   │   ├── engine.py          # SQLModel engine factory
│       │   │   ├── session.py         # SessionLocal / async session maker
│       │   │   └── orm/               # SQLModel ORM models (NOT Pydantic)
│       │   │       ├── __init__.py
│       │   │       └── *.py           # One file per ORM entity
│       │   ├── repositories/          # Protocol implementations (@attrs.define)
│       │   │   ├── __init__.py
│       │   │   └── *.py               # One file per repo (user_repo.py)
│       │   └── mappers/               # ORM ↔ Pydantic pure functions
│       │       ├── __init__.py
│       │       └── *.py               # One file per mapper (user_mapper.py)
│       │
│       ├── api/                       # FastAPI HTTP layer
│       │   ├── __init__.py
│       │   ├── deps.py                # Dependency injection wiring
│       │   ├── v1/
│       │   │   ├── __init__.py
│       │   │   ├── router.py          # APIRouter aggregation
│       │   │   └── *.py               # One file per resource (users.py, orders.py)
│       │   └── exceptions.py          # HTTP exception handlers for returns.Failure
│       │
│       └── lib/                       # Shared returns helpers
│           ├── __init__.py
│           └── *.py                   # result_utils.py, validators.py, etc.
│
└── tests/
    ├── conftest.py                    # pytest fixtures
    ├── unit/
    │   └── domain/
    │       └── test_*.py
    └── integration/
        └── infrastructure/
            └── test_*.py
```

## Structural Rules

- **Create every directory even if empty.** If a project has no custom enums, still create `domain/enums/__init__.py`. If there is no need for mappers yet, still create `infrastructure/mappers/__init__.py`. This preserves consistency and makes future expansion zero-friction.
- **Never place SQLModel ORM models in `domain/`**. ORM models live exclusively under `infrastructure/db/orm/`.
- **Never place FastAPI routers in `domain/`**. HTTP layer lives exclusively under `api/`.
- **Never place business logic in `infrastructure/`**. Side-effect code (DB, HTTP, files) goes here; decisions go in `domain/services/`.
- **One entity per file.** `domain/models/user.py` contains `UserCreate`, `UserResponse`, `UserUpdate`. Do not lump unrelated entities into a single `models.py`.
- **Placeholder modules:** When a layer is not yet needed, create a minimal module with a docstring explaining its purpose and a `pass` or `...` body. Example: `infrastructure/mappers/__init__.py` may contain only a docstring.
