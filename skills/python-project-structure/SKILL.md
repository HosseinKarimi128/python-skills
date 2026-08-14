---
name: python-project-structure

description: >
  Feature-oriented FastAPI / backend project structure with explicit application,
  domain, and infrastructure boundaries. Use this layout for normal production
  backends unless the user explicitly requests a smaller prototype.
---

# Python Project Structure

Use a feature-oriented structure for normal FastAPI backends. Organize code by
business capability first, while keeping the architectural boundary visible:

```text
api            -> HTTP transport and presentation
application    -> use cases and workflow orchestration
domain         -> business rules, entities, and ports
infrastructure -> database and external-service implementations
```

The application layer is intentional. It answers "what does the system do for
this use case?" and coordinates repositories, domain rules, transactions, and
external gateways. It must not know about FastAPI, HTTP, SQLModel, or vendor SDKs.

## Default Layout

```text
myapp/
├── pyproject.toml
├── Dockerfile
├── compose.yaml
├── entrypoint.sh
├── alembic.ini
├── .env
├── .env.prod
├── assets/
├── postman/
│   ├── collections/
│   └── environments/
├── migrations/
│   ├── env.py
│   ├── script.py.mako
│   └── versions/
├── logs/
│   └── .gitkeep
├── scripts/
├── docs/
├── src/
│   └── myapp/
│       ├── __init__.py
│       ├── main.py                    # App factory, lifespan, middleware
│       ├── config.py                  # Pydantic Settings and env validation
│       ├── state.py                   # Shared runtime resources
│       ├── error.py                   # Cross-layer error/result translation
│       │
│       ├── api/
│       │   ├── __init__.py
│       │   ├── deps.py                # FastAPI dependency wiring
│       │   ├── router.py              # Top-level router aggregation
│       │   ├── exceptions.py          # HTTP exception handlers
│       │   └── v1/
│       │       ├── __init__.py
│       │       ├── router.py
│       │       ├── users/
│       │       │   ├── __init__.py
│       │       │   ├── routes.py
│       │       │   ├── request.py
│       │       │   └── response.py
│       │       └── orders/
│       │           ├── __init__.py
│       │           ├── routes.py
│       │           ├── request.py
│       │           └── response.py
│       │
│       ├── application/
│       │   ├── __init__.py
│       │   ├── ports/                 # Cross-feature application contracts
│       │   │   ├── __init__.py
│       │   │   └── unit_of_work.py
│       │   ├── users/
│       │   │   ├── __init__.py
│       │   │   ├── commands.py        # Use-case input models
│       │   │   ├── create.py          # Register/create workflow
│       │   │   └── get.py             # Read workflow
│       │   └── orders/
│       │       ├── __init__.py
│       │       ├── commands.py
│       │       ├── create.py
│       │       └── cancel.py
│       │
│       ├── domain/
│       │   ├── __init__.py
│       │   ├── shared/                # Truly cross-feature concepts only
│       │   │   ├── __init__.py
│       │   │   ├── errors.py
│       │   │   └── ids.py
│       │   ├── users/
│       │   │   ├── __init__.py
│       │   │   ├── entity.py
│       │   │   ├── value_objects.py
│       │   │   ├── enums.py
│       │   │   ├── errors.py
│       │   │   └── repository.py       # User persistence port
│       │   └── orders/
│       │       ├── __init__.py
│       │       ├── entity.py
│       │       ├── value_objects.py
│       │       ├── enums.py
│       │       ├── errors.py
│       │       └── repository.py
│       │
│       ├── infrastructure/
│       │   ├── __init__.py
│       │   ├── persistence/
│       │   │   ├── __init__.py
│       │   │   ├── db.py              # Engine/session factory
│       │   │   ├── uow.py             # Concrete transaction boundary
│       │   │   ├── models/
│       │   │   │   ├── user.py         # ORM models
│       │   │   │   └── order.py
│       │   │   ├── repositories/
│       │   │   │   ├── user.py         # Repository port implementations
│       │   │   │   └── order.py
│       │   │   └── mappers/
│       │   │       ├── user.py
│       │   │       └── order.py
│       │   └── gateways/
│       │       ├── __init__.py
│       │       ├── payments.py         # External-service adapter
│       │       └── email.py
│       │
│       └── lib/                       # Generic, project-independent helpers
│           └── __init__.py
│
└── tests/
    ├── conftest.py
    ├── unit/
    │   ├── domain/
    │   └── application/
    ├── integration/
    │   └── infrastructure/
    └── e2e/
```

Create only the features and layers needed by the project. For a normal
production backend, keep the top-level boundaries and add placeholder modules
only when they make the architecture clearer. For a deliberately small
prototype, the user may opt into a flatter layout.

## Layer Responsibilities

### `api/` — HTTP Boundary

The API layer translates between HTTP and application inputs/outputs.

- `routes.py` contains thin FastAPI endpoints and route declarations.
- `request.py` contains HTTP request schemas, query parameters, and path models.
- `response.py` contains HTTP response schemas and serialization concerns.
- `deps.py` builds application dependencies from configuration and request context.
- `exceptions.py` maps typed application/domain failures to HTTP responses.
- `router.py` aggregates versioned and feature routers.

An endpoint should validate/extract input, create an application command or
query, call one use case, and translate its result. It should not coordinate
multiple repositories, manage transactions, call vendor SDKs, or contain
business decisions.

### `application/` — Use Cases and Orchestration

The application layer represents actions the system performs: `create_user`,
`get_user`, `place_order`, `cancel_order`, and similar workflows.

An application use case may:

- load data through repository ports;
- call domain constructors, methods, and pure domain services;
- enforce workflow-level authorization or sequencing;
- coordinate multiple features or repositories;
- call external-service ports;
- open, commit, and roll back a Unit of Work;
- return a typed success or failure result.

It must not import FastAPI, SQLModel/SQLAlchemy, HTTP clients, vendor SDKs,
credentials, or concrete infrastructure adapters.

Application code is not necessarily pure. Its job is orchestration. Keep pure
business calculations and invariants in `domain/`.

Use one module per use case when the workflow is substantial. Keep commands,
queries, and result types near their feature instead of collecting all services
in one global `services.py`.

### `domain/` — Business Concepts and Rules

The domain is framework- and I/O-independent. It contains the rules that must
remain true regardless of whether the system is called through HTTP, a worker,
or a command-line process.

Feature-local domain modules may contain:

- entities and aggregates in `entity.py`;
- value objects in `value_objects.py`;
- exhaustive enums and state values in `enums.py`;
- typed domain failures in `errors.py`;
- repository and gateway ports owned by the feature;
- pure domain services for rules that do not belong to one entity.

Do not place ORM models, Pydantic HTTP schemas, FastAPI objects, vendor DTOs,
credentials, or concrete clients here. Domain code may define `Protocol`
interfaces, but it must not import their implementations.

Prefer feature-local enums, errors, and protocols. A global `domain/enums/` or
`domain/protocols/` directory is allowed only for concepts genuinely shared by
multiple features. The module that needs a port should own that port; the
infrastructure module implements it.

### `infrastructure/` — Technical Implementations

Infrastructure is the side-effect boundary. It contains concrete database,
network, file, queue, and third-party integrations.

- `persistence/db.py` creates the engine and session factory.
- `persistence/models/` contains ORM/persistence shapes only.
- `persistence/repositories/` implements domain repository ports.
- `persistence/mappers/` translates ORM rows to domain objects and back.
- `persistence/uow.py` implements the application Unit of Work and transaction
  boundary.
- `gateways/` implements external-service ports and normalizes vendor DTOs.

Keep database transactions and external API calls separate by default. If a
workflow needs reliable cross-system coordination, use an explicit outbox,
saga, idempotency, or retry design rather than hiding the problem in UoW.

## Ports and Unit of Work

Define abstractions at the boundary that consumes them:

```text
application use case
        ↓ depends on
domain/application port
        ↑ implemented by
infrastructure adapter
```

Feature repository ports normally live next to the feature:

```text
domain/orders/repository.py       # protocol
infrastructure/persistence/repositories/order.py  # implementation
```

The Unit of Work contract may live in `application/ports/unit_of_work.py` when
it primarily serves use-case orchestration. A domain-owned `domain/uow.py` is
also valid when the transaction abstraction is part of the domain contract.
Choose one owner and keep the concrete implementation in infrastructure.

The UoW owns transaction scope and exposes the repositories needed by a
workflow. It controls `commit()` and `rollback()`. It should not become a bag
for unrelated external gateways, configuration, or business rules.

## Dependency Direction

```text
api ───────────────► application ───────────────► domain
 │                         │                       ▲
 │                         │                       │
 └──── wiring ─────► infrastructure ──────────────┘
```

More precisely:

- `api` may import application commands, use cases, and result translators.
- `application` may import domain objects and ports.
- `domain` imports no application, API, or infrastructure modules.
- `infrastructure` imports the ports and domain types it implements.
- `main.py`, `state.py`, and `api/deps.py` are composition-root code and may
  assemble concrete infrastructure objects.

## Feature Organization Rules

- Find a feature's API, use cases, domain concepts, and adapters by following
  the same feature name across layers.
- Do not create a global `services.py` for unrelated workflows.
- Do not create a global `models.py` containing API, domain, and ORM models.
- Keep HTTP DTOs in `api/`, application commands/results in `application/`,
  domain models in `domain/`, and ORM models in `infrastructure/`.
- Use shared modules only when ownership is genuinely cross-feature. Avoid a
  vague `common/` or `utils/` dumping ground.
- One concept or cohesive use case per file. Split a file when its reason to
  change becomes unclear.

## Tests

```text
tests/
├── unit/
│   ├── domain/       # No database, network, FastAPI, or real clock
│   └── application/  # Fake ports/UoW; test workflows and failure paths
├── integration/
│   └── infrastructure/  # Real or controlled DB/gateway boundaries
└── e2e/              # Running API tested as an outside client
```

Unit-test domain rules without I/O. Application tests use fake repositories,
fake gateways, and a fake UoW. Integration tests verify repository mappings,
transactions, and gateway adapters. End-to-end tests use the public API rather
than importing application internals.

## Operational Files

- `Dockerfile` builds and runs the application image.
- `compose.yaml` describes local or production service orchestration as the
  project requires.
- `entrypoint.sh` performs startup work and uses `exec` for the final process.
- `.env` and `.env.prod` contain templates/placeholders, never real secrets.
- `migrations/` contains Alembic migrations only.
- `scripts/` contains black-box scripts that call the running API externally.
- `postman/` contains shared manual API collections and environments.
- `assets/` contains local or operational files intentionally excluded from the
  application image.
- `logs/` contains local runtime output and is normally gitignored.
- `docs/` contains human-facing architecture, API, deployment, and runbook docs.

## Structural Rules

- Keep handlers thin and use cases explicit.
- Never put ORM models in `domain/` or `application/`.
- Never put FastAPI request/response schemas in `domain/`.
- Never put business workflow orchestration in `api/` or `infrastructure/`.
- Never import concrete infrastructure classes from domain code.
- Never leak vendor DTOs beyond the adapter that normalizes them.
- Keep pure domain rules free of I/O and framework imports.
- Define repository/gateway ports separately from their implementations.
- Keep the Unit of Work focused on transaction scope; do not treat it as a
  universal service container.
- Use typed errors/results at layer boundaries. Do not use generic exceptions
  as routine business outputs.
- Use enums for meaningful finite state and category values; do not scatter
  raw category strings throughout the domain.
- Do not commit generated logs or real secrets.
- Keep production images lean; copy only assets explicitly required at runtime.
- Add an `application/` layer when handlers or domain services coordinate
  repositories, transactions, or external services. Do not add it merely as an
  empty ceremony for a tiny prototype.

## When the Application Layer Is Missing

Without an explicit application layer, orchestration usually leaks into one of
two places:

```text
fat API handlers       or       impure domain services
```

Over time this causes duplicated workflows across HTTP, workers, and tests,
transaction management mixed with HTTP concerns, domain code coupled to the
database, harder unit tests, and unclear ownership of authorization and error
translation. The application layer prevents this by giving each system action
one reusable home while leaving domain rules independent of the delivery
mechanism.

The layer is a design responsibility, not a mandatory amount of ceremony. In a
small project it may be a few modules. In a larger project it should be
feature-oriented and explicit.
