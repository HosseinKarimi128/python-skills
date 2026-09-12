---
name: python-project-structure

description: >
  PostgreSQL-first FastAPI/backend project structure. Use SQL files as first-class
  source, keep Python focused on HTTP, workflow orchestration, domain types, and
  external gateways. Avoid ORM/repository ceremony unless a concrete project
  requirement justifies it.
---

# Python Project Structure

Use a PostgreSQL-first backend architecture with minimal ceremony.

Core principles:

```text
api       -> HTTP boundary and Pydantic request/response models
workflow  -> non-trivial orchestration only
domain    -> enums, dataclasses, and Protocols
db        -> generic PostgreSQL runtime machinery
gateway   -> concrete external-system adapters
sql       -> all SQL source: runtime queries, migrations, functions, views, triggers
```

Do not create layers merely because they are common in Clean Architecture. A layer
or abstraction must solve a real problem in the current project.

## Default Layout

```text
project/
├── pyproject.toml
├── Dockerfile
├── compose.yaml
├── entrypoint.sh
├── .env
├── .env.prod
│
├── sql/
│   ├── migrations/
│   │   ├── 0001_initial.sql
│   │   ├── 0002_users.sql
│   │   └── 0003_orders.sql
│   ├── users/
│   │   ├── get.sql
│   │   ├── create.sql
│   │   ├── update.sql
│   │   ├── delete.sql
│   │   └── search.sql
│   ├── orders/
│   │   ├── get.sql
│   │   ├── create.sql
│   │   ├── cancel.sql
│   │   └── list.sql
│   ├── reports/
│   │   └── sales_performance.sql
│   ├── functions/
│   ├── views/
│   ├── triggers/
│   └── seeds/
│
├── src/
│   └── myapp/
│       ├── __init__.py
│       ├── main.py
│       ├── config.py
│       ├── state.py
│       ├── error.py
│       │
│       ├── api/
│       │   ├── deps.py
│       │   ├── router.py
│       │   ├── exceptions.py
│       │   └── v1/
│       │       ├── users.py
│       │       ├── orders.py
│       │       ├── reports.py
│       │       └── auth.py
│       │
│       ├── workflow/
│       │   ├── checkout_order.py
│       │   ├── register_with_invitation.py
│       │   ├── process_refund.py
│       │   └── sync_catalog.py
│       │
│       ├── domain/
│       │   ├── user.py
│       │   ├── order.py
│       │   ├── payment.py
│       │   └── shared.py
│       │
│       ├── db/
│       │   ├── pool.py
│       │   ├── transaction.py
│       │   ├── sql.py
│       │   └── errors.py
│       │
│       ├── gateway/
│       │   ├── payment.py
│       │   ├── email.py
│       │   ├── storage.py
│       │   └── openai.py
│       │
│       └── lib/
│
├── tests/
│   ├── unit/
│   │   ├── domain/
│   │   └── workflow/
│   ├── integration/
│   │   ├── sql/
│   │   ├── db/
│   │   └── gateway/
│   └── e2e/
│
├── scripts/
└── docs/
```

Keep directories flat by default. Split a module into a package only when real
size or responsibility growth makes a flat module difficult to understand.

## Execution Paths

There are two normal execution paths.

### Simple database operation

For straightforward CRUD, filtering, reporting, or other database-oriented work:

```text
API -> db/sql -> .sql file -> PostgreSQL
```

Do not force a workflow hop for a simple query.

Examples:

```text
GET    /users/{id}     -> api/v1/users.py -> sql/users/get.sql
POST   /users          -> api/v1/users.py -> sql/users/create.sql
PATCH  /users/{id}     -> api/v1/users.py -> sql/users/update.sql
GET    /sales/report   -> api/v1/reports.py -> sql/reports/sales_performance.sql
```

SQL complexity alone does not justify a workflow. A large report query with CTEs,
window functions, joins, or aggregation may still execute directly from the API.

### Non-trivial workflow

Use `workflow/` only when the operation coordinates multiple steps, business
rules, transactions, domains, or external systems:

```text
API -> workflow -> domain / db/sql / gateway
```

Examples:

```text
checkout order
register with invitation
refund payment
sync external catalog
multi-step transaction sequencing
cross-system coordination
```

A workflow module should describe an action, for example:

```text
workflow/checkout_order.py
workflow/process_refund.py
workflow/register_with_invitation.py
```

Keep `workflow/` flat by default. Group workflows only when the number of related
workflows makes navigation difficult.

## `api/` — HTTP Boundary

The API layer owns HTTP concerns.

It may contain:

- FastAPI routers and handlers;
- Pydantic request models;
- Pydantic response models;
- query/path parameter models;
- authentication extraction;
- HTTP error translation.

Keep one cohesive API capability per file by default:

```text
api/v1/users.py
api/v1/orders.py
api/v1/reports.py
```

A module may contain its closely related request/response schemas and routes.
Do not automatically split every feature into `routes.py`, `request.py`, and
`response.py`.

Split only when the module becomes genuinely difficult to navigate.

Simple database operations may call the generic SQL executor directly from the
API layer. Do not introduce a workflow module that merely forwards arguments to
one SQL file.

## `workflow/` — Orchestration Only

`workflow/` contains non-trivial application workflows.

Use it when an operation does more than a straightforward database operation.
Typical reasons include:

- multiple ordered steps;
- multiple SQL operations that form one application workflow;
- domain decisions or domain Protocols;
- external gateway calls;
- transaction sequencing;
- cross-feature coordination;
- idempotency, retries, or compensation logic across systems.

Do not place ordinary CRUD here.

Do not create workflow wrappers like:

```text
workflow/get_user.py
workflow/create_user.py
workflow/update_user.py
```

when they only execute one SQL file.

## `domain/` — Pure Domain Types

The domain layer is framework-free and I/O-free.

Domain modules are flat by default:

```text
domain/user.py
domain/order.py
domain/payment.py
```

A domain module may define three kinds of classes:

1. `Enum` / `StrEnum`
2. `dataclass`
3. `Protocol`

A module does not need to contain all three.

Example:

```python
from dataclasses import dataclass
from enum import StrEnum
from typing import Protocol


class OrderStatus(StrEnum):
    PENDING = "pending"
    PAID = "paid"
    CANCELLED = "cancelled"


@dataclass(frozen=True, slots=True)
class Order:
    id: UUID
    total: Decimal
    status: OrderStatus


class PaymentProvider(Protocol):
    async def charge(self, order: Order) -> str: ...
```

The domain must not contain:

```text
Pydantic
FastAPI
SQL
psycopg
SQLAlchemy / SQLModel
HTTP clients
vendor SDKs
```

Pydantic belongs at system boundaries such as `api/` and `gateway/`.

Do not invent domain objects for plain CRUD when no domain rule or workflow needs
them.

Protocols exist only when a real workflow benefits from an abstract contract.
Do not add repository Protocols merely to hide PostgreSQL.

## `db/` — Generic PostgreSQL Runtime Machinery

Keep `db/` small and generic.

```text
db/
├── pool.py
├── transaction.py
├── sql.py
└── errors.py
```

Responsibilities:

- `pool.py`: connection pool creation and lifecycle;
- `transaction.py`: generic transaction helpers;
- `sql.py`: load and execute SQL source files;
- `errors.py`: database error normalization when needed.

Do not place user-, order-, or feature-specific database logic in `db/`.

The SQL executor should expose a very small API, conceptually:

```python
await sql.one(conn, "users/get.sql", params)
await sql.many(conn, "users/search.sql", params)
await sql.execute(conn, "users/delete.sql", params)
```

The implementation may support typed row factories or explicit conversion at the
caller boundary, but database behavior must remain in SQL files.

Do not execute `psql` as a subprocess for normal HTTP requests. Use a PostgreSQL
client/connection pool such as psycopg. PostgreSQL still performs the query;
Python only sends the SQL and receives results.

## `sql/` — First-Class SQL Source

All runtime SQL lives in `.sql` files.

Do not embed application SQL strings in Python modules by default.

Example:

```sql
-- sql/users/get.sql
SELECT
    id,
    name,
    email,
    created_at
FROM users
WHERE id = %(user_id)s;
```

Organize runtime SQL by database/business capability:

```text
sql/users/
sql/orders/
sql/reports/
```

Keep PostgreSQL-owned objects in dedicated directories:

```text
sql/migrations/
sql/functions/
sql/views/
sql/triggers/
sql/seeds/
```

### Migrations

Use SQL migrations as the authoritative schema history.

Do not use ORM-generated schema definitions as the source of truth.
Do not require Alembic Python migrations for this architecture.

A migration runner may still track and execute versioned `.sql` files.

### PostgreSQL-owned logic

Deliberately use PostgreSQL for data-centric behavior:

```text
NOT NULL
CHECK
UNIQUE
FOREIGN KEY
EXCLUDE

joins
CTEs
window functions
aggregation

INSERT / UPDATE / DELETE
RETURNING

transactions
row/advisory locks
concurrency-sensitive operations

views
materialized views
functions
triggers when appropriate

indexes
partial indexes
expression indexes
GIN / GiST

JSONB
full-text search
```

Prefer atomic database operations over Python read-modify-write sequences when
PostgreSQL can enforce the operation safely.

Example:

```sql
UPDATE accounts
SET balance = balance - %(amount)s
WHERE id = %(id)s
  AND balance >= %(amount)s
RETURNING *;
```

Do not fetch rows into Python merely to reproduce relational work that
PostgreSQL handles naturally.

## `gateway/` — External Systems

`gateway/` contains concrete adapters for systems outside the application:

```text
gateway/payment.py
gateway/email.py
gateway/storage.py
gateway/openai.py
```

Examples include:

- payment providers;
- email/SMS providers;
- OpenAI or other AI services;
- object storage;
- third-party HTTP APIs.

Gateway modules may use Pydantic for external request/response DTOs.

When a workflow needs an abstract external contract, define the Protocol in the
relevant domain module and implement it in `gateway/`.

Do not create a separate `ports/` directory by default.

## Dependency Guidance

Simple path:

```text
api -> db/sql -> SQL -> PostgreSQL
```

Workflow path:

```text
                    domain
                   ▲      ▲
                   │      │
api -> workflow ---┘      │
         │                │
         ├-> db/sql -> SQL -> PostgreSQL
         │
         └-> gateway -----┘ -> external system
```

The architecture intentionally does not hide PostgreSQL behind repositories.
PostgreSQL is a first-class application dependency.

## Things Intentionally Absent

Do not add these by default:

```text
ORM
SQLAlchemy ORM models
SQLModel table models
repository layer
repository Protocols
persistence mappers
persistence model duplicates
query layer in Python
repository-style Unit of Work
mandatory workflow/application hop
ports directory
Alembic Python migrations
```

Introduce an abstraction only when a concrete project requirement justifies it.

## Structural Rules for Agents

When creating or refactoring a project, apply these rules in order:

1. Start flat.
2. Keep API modules flat by capability unless they become too large.
3. Route simple CRUD/query/report endpoints directly from API to SQL files.
4. Create a workflow only for real orchestration.
5. Keep workflow modules flat and action-named.
6. Keep domain modules flat and framework-free.
7. Domain class kinds are `Enum`, `dataclass`, and `Protocol` only unless the user explicitly requests otherwise.
8. Keep Pydantic models at API/gateway boundaries.
9. Keep all runtime SQL in `.sql` files.
10. Keep schema history in SQL migrations.
11. Prefer PostgreSQL constraints and atomic operations for data integrity.
12. Prefer joins/aggregation/CTEs in PostgreSQL over Python-side relational processing.
13. Keep `db/` generic and feature-agnostic.
14. Keep external vendor details in `gateway/`.
15. Do not invent repository/UoW/mapper/port abstractions for hypothetical replaceability.
16. Add hierarchy only when real complexity makes the flat form hard to maintain.

## Decision Checklist

Before adding a Python layer or abstraction, ask:

```text
Is this a simple database operation?
    -> API -> SQL

Does this operation coordinate multiple steps, domain decisions, transactions,
or external systems?
    -> API -> workflow

Is this a pure business concept or contract used by a workflow?
    -> domain

Is this generic PostgreSQL execution/lifecycle machinery?
    -> db

Is this a concrete external-system implementation?
    -> gateway

Is this relational/data logic, schema, migration, reporting, or atomic mutation?
    -> SQL/PostgreSQL
```

If no concrete complexity requires another layer, do not add one.
