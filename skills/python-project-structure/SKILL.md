---
name: python-project-structure

description: >
  Mandatory FastAPI / backend project directory layout and structural rules.
  Every project must follow this layout unless the user explicitly requests a smaller prototype.
  Create the core directories consistently so domain logic, infrastructure I/O, and API boundaries stay separated.
---

# Python Project Structure

## Directory Layout

Use the following directory layout for every FastAPI / backend project. If a project spec does not require a given layer yet, still create the corresponding directory with a minimal `__init__.py` or placeholder module when doing so preserves architectural consistency.

```text
myapp/
├── pyproject.toml
├── Dockerfile                          # Production application image
├── compose.yaml                        # Production Docker Compose stack
├── entrypoint.sh                       # Container startup script
├── alembic.ini                         # Alembic configuration
├── .env                                # Local development environment variables
├── .env.prod                           # Production environment variables/template
├── assets/                             # Files intentionally excluded from app image
│   └── ...
├── postman/                            # Postman collections and environments
│   ├── collections/
│   │   └── *.json
│   └── environments/
│       └── *.json
├── migrations/                         # Alembic migration environment
│   ├── env.py
│   ├── script.py.mako
│   └── versions/
│       └── *.py
├── logs/                               # Runtime logs, usually gitignored
│   └── .gitkeep
├── scripts/                            # Bash black-box API workflow scripts
│   └── *.sh
├── docs/                               # Project documentation
│   └── *.md / .excalidraw
├── src/
│   └── myapp/
│       ├── __init__.py
│       ├── main.py                    # FastAPI app factory, lifespan, middleware
│       ├── config.py                  # Pydantic Settings (env validation)
│       │
│       ├── domain/                    # Pure logic, zero I/O
│       │   ├── __init__.py
│       │   ├── models/                # Internal domain value objects
│       │   │   ├── __init__.py
│       │   │   └── *.py               # One file per domain concept
│       │   ├── schemas/               # App-owned I/O contracts, not vendor DTOs
│       │   │   ├── __init__.py
│       │   │   └── *.py               # Request/Response payload shapes
│       │   ├── enums/                 # Exhaustive enums for match-case
│       │   │   ├── __init__.py
│       │   │   └── *.py               # One file per enum
│       │   ├── protocols/             # @runtime_checkable traits for outbound I/O
│       │   │   ├── __init__.py
│       │   │   ├── repository.py      # Repository[T, ID] protocol for persistence
│       │   │   ├── *_gateway.py       # External service protocols
│       │   │   └── unit_of_work.py    # UnitOfWork protocol for DB transactions
│       │   ├── services/              # Pure functions, no classes
│       │   │   ├── __init__.py
│       │   │   └── *.py               # One file per service
│       │   └── errors/                # Explicit error types
│       │       ├── __init__.py
│       │       ├── base.py            # DomainError base
│       │       └── *.py               # One file per error variant
│       │
│       ├── infrastructure/            # I/O boundaries, side effects only here
│       │   ├── __init__.py
│       │   ├── db/
│       │   │   ├── __init__.py
│       │   │   ├── engine.py          # SQLModel engine factory
│       │   │   ├── session.py         # SessionLocal / async session maker
│       │   │   └── orm/               # SQLModel ORM models
│       │   │       ├── __init__.py
│       │   │       └── *.py           # One file per ORM entity
│       │   ├── repositories/          # DB protocol implementations
│       │   │   ├── __init__.py
│       │   │   └── *.py               # user_repo.py, order_repo.py
│       │   ├── gateways/              # External API / third-party service adapters
│       │   │   ├── __init__.py
│       │   │   └── *.py               # stripe_gateway.py, sendgrid_gateway.py
│       │   ├── mappers/               # ORM/API DTO ↔ app-owned models/schemas
│       │   │   ├── __init__.py
│       │   │   └── *.py               # user_mapper.py, stripe_mapper.py
│       │   └── unit_of_work.py        # SQLUnitOfWork transaction boundary
│       │
│       ├── api/                       # FastAPI HTTP layer
│       │   ├── __init__.py
│       │   ├── deps.py                # Dependency injection wiring
│       │   ├── schemas/               # HTTP-specific request/response schemas
│       │   │   ├── __init__.py
│       │   │   └── *.py
│       │   ├── v1/
│       │   │   ├── __init__.py
│       │   │   ├── router.py          # APIRouter aggregation
│       │   │   └── *.py               # One file per resource
│       │   └── exceptions.py          # HTTP exception handlers for Failure
│       │
│       └── lib/
│           ├── __init__.py
│           └── ...                    # Pure shared helpers
│
└── tests/
    ├── conftest.py
    ├── unit/
    │   └── domain/
    │       └── test_*.py
    └── integration/
        └── infrastructure/
            └── test_*.py
```

## Root Directory Purpose and Contents

| Directory / File | Purpose | What goes inside |
|---|---|---|
| `pyproject.toml` | Project metadata, dependencies, tool configuration, and package settings. | Runtime dependencies, dev dependencies, pytest/ruff/ty settings. |
| `Dockerfile` | Production application image definition. | Multi-stage build if useful, dependency install, non-root runtime user, app startup command. |
| `compose.yaml` | Production Docker Compose stack. | App service, database service, networks, volumes, healthchecks, restart policy. |
| `entrypoint.sh` | Container startup script. | Runtime setup, optional migration command, then `exec` the app process. |
| `alembic.ini` | Alembic CLI configuration. | Database URL config references, script location, logging config. |
| `.env` | Local development environment variables. | Local database URL, debug flags, local service settings. |
| `.env.prod` | Production environment variables or production template. | Production variable names and deployment-time values/placeholders. |
| `assets/` | Files intentionally excluded from the app image. | Large local artifacts, design files, raw uploads, seeds, private operational assets, temporary import/export files. |
| `postman/` | Postman assets for manual API exploration and shared request workflows. | Collections, environments, example request suites. |
| `migrations/` | Alembic migration environment and generated migration revisions. | `env.py`, `script.py.mako`, `versions/*.py`. |
| `logs/` | Local runtime log output. This directory should usually be gitignored except for `.gitkeep`. | `.gitkeep`; generated `.log` files during local/dev runs. |
| `scripts/` | Bash scripts for black-box API workflows and operational checks. These scripts interact with the running app from the outside. | API smoke tests, end-to-end request flows, local setup helpers. |
| `docs/` | Project documentation for architecture, API behavior, deployment notes, and operational runbooks. | `architecture.md`, `api.md`, `development.md`, `deployment.md`. |
| `src/` | Application source package. | FastAPI app, domain logic, infrastructure adapters, API routes. |
| `tests/` | Automated test suite. | Unit tests, integration tests, fixtures. |

## Directory Purpose and Contents

### `domain/` — Pure Logic, Zero I/O

The **heart of the application**. Nothing in this directory imports SQLModel, FastAPI, HTTP clients, third-party SDKs, credentials, or any I/O library. It is fully testable with no database, no server, and no network.

| Sub-directory | Purpose | What goes inside |
|---|---|---|
| `domain/models/` | Internal domain value objects. Prefer `@dataclass(frozen=True, slots=True)`. These are not Pydantic and carry no validation overhead. | `UserSummary`, `Money`, `TaskAssignment`, `OrderTotals`. |
| `domain/schemas/` | App-owned I/O contracts used by protocols and services. These may be Pydantic models, but they must not be raw vendor DTOs. | `PaymentRequest`, `PaymentResponse`, `EmailSendRequest`. |
| `domain/enums/` | Exhaustive tagged unions for `match-case` branching. Every state machine, discriminated union, or categorical value is an enum. | `TaskStatus(StrEnum)`, `PaymentStatus(StrEnum)`. |
| `domain/protocols/` | Rust-like traits using `@runtime_checkable Protocol`. They declare what operations exist, not how they are implemented. | `repository.py`, `payment_gateway.py`, `email_gateway.py`, `unit_of_work.py`. |
| `domain/services/` | Pure functions that encode business rules and orchestration. No classes, no side effects. | `create_task(...) -> Result[TaskSummary, TaskError]`. |
| `domain/errors/` | Explicit, structured error types returned inside `Failure`. Every failure mode is a named class. | `TaskNotFoundError`, `PaymentDeclinedError`. |

**Key rules for domain models and schemas:**

- Internal domain models live in `domain/models/`.
- App-owned I/O contracts live in `domain/schemas/`.
- HTTP-only request/response schemas may live in `api/schemas/`.
- External vendor objects must never leak into `domain/`.
- Normalize third-party API payloads into app-owned schemas before passing them into domain services.
- Domain protocols may reference app-owned domain models and schemas, but never ORM models, SDK objects, HTTP responses, or vendor DTOs.

### `infrastructure/` — I/O Boundaries

The **dirty edge** of the application. This is the only place `session.commit()`, HTTP requests, third-party SDK calls, file writes, or external API calls happen. Domain code never imports from here.

| Sub-directory | Purpose | What goes inside |
|---|---|---|
| `infrastructure/db/` | Database connectivity. Engine, session factory, and SQLModel ORM table definitions. | `engine.py`, `session.py`, `orm/user.py`. |
| `infrastructure/db/orm/` | SQLModel table schemas. These are persistence-layer shapes, not API contracts and not domain models. | `UserORM`, `OrderORM`. |
| `infrastructure/repositories/` | Database persistence adapter implementations. Concrete classes satisfy repository protocols from `domain/protocols/`. | `SQLUserRepository`, `SQLOrderRepository`. |
| `infrastructure/gateways/` | External service adapter implementations. Treat third-party API I/O like DB I/O: define the abstraction in `domain/protocols/`, implement the concrete adapter here. | `StripePaymentGateway`, `SendgridEmailGateway`, `S3FileGateway`. |
| `infrastructure/mappers/` | Pure translation functions between ORM/API DTO objects and app-owned domain models or schemas. | `orm_to_user_summary()`, `stripe_payload_to_payment_response()`. |
| `infrastructure/unit_of_work.py` | Database transaction boundary. Holds the database session, injects it into repositories, and controls `commit()` / `rollback()`. | `SQLUnitOfWork`. |

External service gateways usually should not be part of the database `UnitOfWork`. Keep DB transactions and third-party API calls separate unless the project explicitly needs an orchestration pattern such as an outbox, saga, or idempotent retry workflow.

### `api/` — FastAPI HTTP Layer

The **presentation boundary**. Validates incoming JSON, calls domain services, and converts `Result` into HTTP responses. No business logic lives here.

| Sub-directory / File | Purpose | What goes inside |
|---|---|---|
| `api/deps.py` | Dependency injection wiring. FastAPI `Depends()` factories that build repositories, unit-of-work instances, and external gateways from config/request context. | `get_session()`, `get_uow()`, `get_payment_gateway()`. |
| `api/schemas/` | HTTP-specific request/response schemas. Use this when a schema exists only because of the public HTTP API. | `CreateUserBody`, `UserHTTPResponse`. |
| `api/exceptions.py` | HTTP exception handlers. Converts `Failure` objects into `HTTPException` with correct status codes. | `handle_result(result)`. |
| `api/v1/router.py` | Router aggregation. Collects all resource routers under `/v1`. | `router.include_router(users.router, prefix="/users")`. |
| `api/v1/*.py` | Resource endpoints. One file per REST resource. | `users.py`, `tasks.py`, `orders.py`. |

### `lib/`

Small, reusable, **pure shared helpers** that multiple layers may import. They must not depend on project-specific domain, infrastructure, or API code.

Good candidates:

- retry helpers
- logging setup helpers
- clock/id provider helpers
- serialization helpers
- small functional utilities

Avoid putting business logic, framework wiring, repositories, gateways, or project-specific decisions in `lib/`.

## Protocols and Wiring

- `domain/protocols/repository.py` defines generic persistence protocols for database communication.
- `domain/protocols/*_gateway.py` defines external service protocols for third-party API communication.
- `domain/protocols/unit_of_work.py` defines DB transaction behavior with repository properties plus `commit()` and `rollback()`.
- Domain protocols must not import SQLModel, ORM models, HTTP clients, SDK objects, credentials, vendor DTOs, or infrastructure modules.
- `infrastructure/repositories/*.py` implements database repository protocols.
- `infrastructure/gateways/*.py` implements external service gateway protocols.
- `infrastructure/unit_of_work.py` implements the DB transaction protocol.
- `api/deps.py` wires concrete implementations into the application.

All business logic lives in `domain/services/*.py` as pure functions. Domain services know nothing about database details, HTTP details, SDKs, credentials, vendor-specific payloads, or concrete infrastructure classes. They operate on typed app-owned values and chain `Result` / `Option`.

## Operational Assets

- `Dockerfile` defines the production app image. Keep it focused on building and running the application, not storing local-only assets.
- `compose.yaml` defines the production Compose stack. Use it for app/database/service orchestration, networks, volumes, healthchecks, and restart policy.
- `entrypoint.sh` performs container startup work, then must `exec` the final app process.
- `.env` is for local development values.
- `.env.prod` is for production values or deployment-time placeholders. Do not commit real secrets.
- `assets/` is for files that should stay outside the app image.
- `postman/` is for shared manual API collections and environment files.
- `scripts/` is for Bash black-box API workflow scripts that call the running service over HTTP. These scripts must not import application internals from `src/`.
- `migrations/` is for Alembic database migrations only. Application runtime code must not live here.
- `logs/` is for local runtime output. Commit only `.gitkeep` or documented sample logs when explicitly useful.
- `docs/` is for human-facing project documentation, not generated runtime output.

## Tests

| Sub-directory | Purpose | What goes inside |
|---|---|---|
| `tests/conftest.py` | Shared fixtures. | In-memory SQLite engine, `AsyncSession`, fake UoW, fake gateways. |
| `tests/unit/domain/` | Pure logic tests with no database and no network. | Service tests using fake repositories and fake gateways. |
| `tests/integration/infrastructure/` | Boundary tests against real or controlled dependencies. | Repository tests with SQLite; gateway tests with mocked HTTP/SDK boundaries. |

## Structural Rules

- **Create core directories consistently.** For normal backend projects, create the listed structure even if some directories only contain placeholders. For tiny prototypes, reduce ceremony only when the user explicitly asks.
- **Never place SQLModel ORM models in `domain/`.** ORM models live exclusively under `infrastructure/db/orm/`.
- **Never place FastAPI routers in `domain/`.** HTTP layer lives exclusively under `api/`.
- **Never place HTTP clients, third-party SDKs, vendor DTOs, credentials, or concrete API clients in `domain/`.** External service I/O lives under `infrastructure/gateways/` behind protocols from `domain/protocols/`.
- **Never place business logic in `infrastructure/`.** Side-effect code goes there; decisions go in `domain/services/`.
- **Do not put external API gateways inside the DB UnitOfWork by default.** Use separate gateway dependencies unless a real consistency pattern requires otherwise.
- **Do not import application internals from black-box scripts.** Scripts under `scripts/` should test the API as an outside client.
- **Do not commit generated logs.** Keep `logs/.gitkeep` if the directory should exist in git.
- **Do not commit real secrets.** `.env` and `.env.prod` should be templates/placeholders unless the repository is strictly private and the user explicitly asks otherwise.
- **Keep app images lean.** Do not copy `assets/`, `postman/`, `docs/`, generated logs, or black-box workflow artifacts into the production image unless explicitly required.
- **One concept per file.** One entity, protocol, gateway, repository, mapper, or error family per file.
- **Enums are exhaustive.** Every categorical value is an enum in `domain/enums/`. Raw strings for categories are forbidden.
- **Errors are typed.** Every failure mode is a class in `domain/errors/`. Generic `Exception` or `ValueError` as function outputs are forbidden.
- **Placeholder modules are allowed.** When a layer is not yet needed, create a minimal module with a docstring explaining its purpose and a `pass` or `...` body.
