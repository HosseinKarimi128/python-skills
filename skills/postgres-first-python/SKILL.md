---
name: postgres-first-python

description: >
  Canonical production Python backend skill for PostgreSQL-first FastAPI projects.
  Defines project structure, SQL-first architecture, Pydantic I/O boundaries,
  functional workflow rules, typed Result/Maybe control flow, domain modeling,
  singledispatch, logging, testing, and quality gates in one authoritative document.
---

# Postgres-First Python

This is the single authoritative Postgres-First Python skill for backend work in this repository.
It defines both **where code belongs** and **how code must be written**.

Use a PostgreSQL-first, low-ceremony architecture. Keep Python focused on HTTP,
workflow orchestration, domain types, external gateways, configuration, lifecycle,
and typed control flow. Keep relational/data work in SQL and PostgreSQL.

Do not add architecture merely because a pattern is common. Every layer, class,
protocol, abstraction, or package must solve a concrete problem in the current
project.

---

## Core Architecture

```text
api/       HTTP boundary; Pydantic request/response models
workflow/  non-trivial business orchestration; functions only
domain/    data-only Enum, frozen dataclass, and Protocol classes
db/        generic PostgreSQL execution/lifecycle machinery
gateway/   concrete external-system adapters; Pydantic at external I/O
error.py   typed application failure values
sql/       all runtime SQL and SQL migrations
```

There are two normal execution paths.

### Simple database operation

For straightforward CRUD, filtering, lookup, reporting, or other database-oriented
work:

```text
API -> db/sql -> .sql file -> PostgreSQL
```

Do **not** create a workflow merely to forward arguments to one SQL statement.

Examples:

```text
GET    /users/{id}   -> api/v1/users.py   -> sql/users/get.sql
POST   /users        -> api/v1/users.py   -> sql/users/create.sql
PATCH  /users/{id}   -> api/v1/users.py   -> sql/users/update.sql
GET    /sales/report -> api/v1/reports.py -> sql/reports/sales_performance.sql
```

SQL complexity alone does not justify `workflow/`. A 200-line reporting query with
CTEs, joins, window functions, or aggregation can still be `API -> SQL`.

### Non-trivial business workflow

Use `workflow/` only when an operation coordinates meaningful application behavior:

```text
API -> workflow -> domain / db/sql / gateway
```

Typical reasons:

- multiple ordered business steps;
- multiple SQL operations that form one workflow;
- domain decisions;
- external gateway calls;
- cross-feature coordination;
- transaction sequencing;
- idempotency, retries, or compensation across systems.

Examples:

```text
checkout order
register with invitation
process refund
sync external catalog
```

---

## Default Project Layout

Keep the project flat by default and split only when real complexity requires it.

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

Do not create placeholder packages merely for hypothetical future use. Start flat.
Promote a file into a package only when size or responsibility growth makes the flat
form genuinely difficult to maintain.

---

## `api/` — HTTP Boundary

The API layer owns HTTP concerns:

- FastAPI routers and handlers;
- Pydantic request models;
- Pydantic response models;
- path/query parameter models when modeled explicitly;
- authentication extraction;
- HTTP error/result translation.

Keep one cohesive capability per file by default:

```text
api/v1/users.py
api/v1/orders.py
api/v1/reports.py
```

A module may contain its closely related schemas and routes. Do not automatically
split every capability into `routes.py`, `request.py`, and `response.py`.

Simple CRUD/query/report routes may execute SQL directly through `db/sql`.

API executable functions follow the same typed `Result` discipline as other business
functions, but their external boundary types are Pydantic models instead of domain
dataclasses.

Conceptually:

```python
Result[UserResponse, AppError]
```

Use one centralized HTTP Result adapter/decorator to map containers to framework
behavior:

```text
Success(ResponseModel)  -> normal FastAPI response
Failure(NotFound)       -> 404
Failure(InvalidInput)   -> 400/422
Failure(DependencyDown) -> 503
Failure(Unexpected)     -> 500
```

Do not duplicate `Result` unwrapping and error-to-status branching in every route.

---

## `workflow/` — Functions-Only Business Orchestration

`workflow/` contains **functions only**. Never define classes there.

Do not put simple CRUD wrappers in workflow. A workflow module should be action
oriented:

```text
workflow/checkout_order.py
workflow/process_refund.py
workflow/register_with_invitation.py
```

There are only two function categories inside workflow modules.

### Helper functions

Helpers are local computational steps.

Rules:

- name starts with `_`;
- private to the module;
- never imported by other modules;
- computational/transformation focused, not a business workflow;
- may accept primitive values;
- may have a primitive success type;
- return `Result[T, E]` or `Maybe[T]`;
- may use `Maybe[T]` when absence, rather than failure, is the only alternative.

Example:

```python
def _calculate_total(
    subtotal: Decimal,
    tax: Decimal,
) -> Result[Decimal, AppError]:
    ...
```

### Pipeline functions

Pipelines represent business work.

Rules:

- public functions;
- orchestrate helpers and, when useful, other pipeline functions;
- accept business-typed domain values inside the business layer;
- return `Result[SuccessType, FailureType]`;
- `SuccessType` must **not** be primitive;
- pipeline success is a domain dataclass or enum;
- compose steps mainly through `returns` chaining;
- pipeline code should remain linear and readable.

Example:

```python
def checkout_order(
    order: Order,
    payment: PaymentProvider,
) -> Result[CheckoutResult, AppError]:
    ...
```

A Protocol parameter is an abstract contract implemented by a concrete object wired
at the composition root. Workflow code does not instantiate Protocols themselves.

---

## `domain/` — Data and Contracts Only

Keep `domain/` flat by default:

```text
domain/user.py
domain/order.py
domain/payment.py
```

A domain module may define only these class forms:

1. `Enum` / `StrEnum`;
2. `@dataclass(frozen=True, slots=True)`;
3. `Protocol`.

A module does not need all three.

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
    async def charge(
        self,
        order: Order,
    ) -> Result[PaymentResult, AppError]: ...
```

Domain rules:

- no executable free functions;
- no business methods on dataclasses;
- no helper methods;
- no properties for behavior;
- dataclasses are data-only;
- Protocol methods contain signatures only;
- no Pydantic;
- no FastAPI;
- no SQL/psycopg;
- no SQLAlchemy/SQLModel;
- no HTTP clients;
- no vendor SDKs.

Define a Protocol only when a real workflow benefits from an abstract contract. Most
ordinary web-backend operations need no Protocol.

Never create repository Protocols just to hide PostgreSQL.

Do not manufacture domain objects for plain CRUD when no domain rule or workflow
needs them.

---

## `error.py` — Typed Failure Values

Expected failures are values, not exceptions used for business control flow.

Define expected application failure types in the upper-level `error.py`, normally as
frozen slotted dataclasses or enums.

Do not require expected error values to inherit from `Exception`.

```python
@dataclass(frozen=True, slots=True)
class UserNotFound:
    user_id: UUID


@dataclass(frozen=True, slots=True)
class PaymentUnavailable:
    provider: str
```

Every expected failure that a caller may need to handle must have a named typed
variant.

Type aliases/unions may collect related failure values when useful:

```python
type CheckoutError = (
    UserNotFound
    | PaymentUnavailable
    | InvalidOrderState
)
```

Do not return failure strings as the primary failure contract when callers need to
branch on the failure.

---

## Strict Type Discipline

Every function and method must have explicit parameter annotations and an explicit
`->` return annotation.

Avoid:

```text
Any
dict[str, Any]
list[Any]
implicit Optional
weak untyped payloads
ambiguous unions
```

Treat `Any` as a last-resort escape hatch requiring strong justification.

Prefer:

- exact types;
- Pydantic boundary models;
- frozen dataclasses;
- enums;
- Protocols;
- precise type aliases;
- typed containers;
- `object` when the runtime type is intentionally unknown;
- recursive JSON aliases or `TypeAdapter` when appropriate.

Never allow weak third-party typing to leak beyond the boundary. Validate or normalize
it immediately.

Use meaningful business types instead of unstructured dictionaries when data has
business meaning.

Prefer immutable values. Domain dataclasses must default to:

```python
@dataclass(frozen=True, slots=True)
```

Do not use `attrs` by default.

---

## Pydantic — Mandatory at External I/O Boundaries

Pydantic is mandatory for all **external I/O boundaries**, including:

- FastAPI request bodies;
- FastAPI response bodies;
- modeled path/query parameters;
- gateway provider requests;
- gateway provider responses;
- configuration/environment input;
- queue/event payloads;
- file/CLI input;
- any other external payload entering or leaving the process.

Validate external data immediately when it crosses the boundary.

Use `Field`, validators, constrained types, discriminated unions, and exact enums as
needed to make invalid external states unrepresentable.

Pydantic does **not** belong in:

```text
domain/
workflow/
```

Do not create database-interface Pydantic models merely to mirror SQL rows or replace
an ORM model layer. The SQL-first architecture intentionally removes that duplicate
persistence model.

For simple CRUD:

```text
SQL row -> API Pydantic response model
```

For workflows:

```text
SQL row -> domain dataclass, only when a business value is needed
```

Gateway implementations may use Pydantic internally for provider DTOs and then
convert validated data into the domain types required by workflows.

Use `T | None` only when null is valid external data. Do not use raw `None` as a
business failure signal after entering the typed business flow.

---

## `returns` — Default Control-Flow Model

Use the `returns` package for expected success, failure, and absence.

```text
Result[T, E] -> Success(T) | Failure(E)
Maybe[T]     -> Some(T) | Nothing
```

Rules:

- executable business functions return `Result[T, E]`;
- private workflow helpers may return `Maybe[T]` when absence is the only alternative;
- pipeline/API/gateway operations do not return bare business success values;
- never use `None` as an implicit failure signal when `Maybe` expresses the contract;
- callers should compose rather than repeatedly unwrap.

Prefer fluent operations such as:

```text
.map(...)
.bind(...)
.lash(...)
.alt(...)
.value_or(...)
```

Prefer fluent chained composition over match-first Result handling when the flow can
remain linear.

Use `match` when there is a genuine finite business branch, especially enum values.
Do not use `match` merely to unwrap every `Result`/`Maybe`.

Example formatting:

```python
result
.map(step_one)
.bind(step_two)
.lash(recover)
```

---

## Type-Driven Dispatch with `functools.singledispatch`

Prefer `@singledispatch` when business behavior naturally varies by the runtime class
of the **first domain argument**.

This is the preferred Python approximation of Elixir-style type-based function
clauses.

```python
@singledispatch
def calculate_price(
    item: object,
) -> Result[Price, AppError]:
    return Failure(
        UnsupportedDomainType(
            type_name=type(item).__name__,
        ),
    )


@calculate_price.register
def _(
    item: PhysicalProduct,
) -> Result[Price, AppError]:
    ...


@calculate_price.register
def _(
    item: SubscriptionProduct,
) -> Result[Price, AppError]:
    ...
```

Rules:

- prefer `singledispatch` over long `isinstance` chains when the behavior is truly
  type-driven;
- the default implementation returns a typed `Failure`;
- do not raise `NotImplementedError` for an expected unsupported domain type;
- dispatch is only on the first argument's runtime class;
- do not use `singledispatch` to distinguish members of one Enum—branch explicitly on
  the enum value;
- registered implementations remain functions;
- do not introduce service classes for dispatched behavior.

The conventional registered function name `_` is a singledispatch implementation,
not a reusable workflow helper, even though its name begins with `_`.

---

## Functional Collection Processing

Avoid imperative `for` loops when the operation is clearer as a declarative iterator
transformation.

Prefer:

- `map`;
- `filter`;
- generators/comprehensions when clearer;
- stdlib `itertools` for lazy composition;
- `returns.iterables.Fold` when composing collections containing `Result`/`Maybe`.

Use fluent `.iter().map().filter()` syntax only when the concrete iterable abstraction
used by the project actually provides that API. Python's stdlib `itertools` does not
provide Rust-style method chaining by itself.

Do not replace a clear simple loop with a harder-to-read functional expression merely
for stylistic purity.

---

## `db/` — Generic PostgreSQL Runtime Machinery

Keep `db/` small, generic, and feature-agnostic:

```text
db/
├── pool.py
├── transaction.py
├── sql.py
└── errors.py
```

Responsibilities:

- `pool.py`: PostgreSQL pool creation/lifecycle;
- `transaction.py`: generic transaction helpers;
- `sql.py`: load and execute SQL source files;
- `errors.py`: DB boundary normalization when useful.

Do not place feature-specific SQL or user/order business logic in `db/`.

The generic SQL runtime API should remain intentionally small, conceptually:

```python
await sql.one(
    conn,
    "users/get.sql",
    params,
)

await sql.many(
    conn,
    "users/search.sql",
    params,
)

await sql.execute(
    conn,
    "users/delete.sql",
    params,
)
```

Do not execute `psql` as a subprocess for normal HTTP requests. Use a PostgreSQL
client/connection pool such as psycopg. PostgreSQL executes the query; Python only
loads/sends the SQL and receives the result.

---

## `sql/` — First-Class Source Code

All runtime SQL lives in `.sql` files.

Do not embed application SQL strings throughout Python modules by default.

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

Organize runtime SQL by capability:

```text
sql/users/
sql/orders/
sql/reports/
```

Keep PostgreSQL-owned definitions in dedicated locations:

```text
sql/migrations/
sql/functions/
sql/views/
sql/triggers/
sql/seeds/
```

### SQL migrations

SQL migrations are the authoritative schema history.

Do not use ORM model definitions as the schema source of truth.
Do not require Alembic Python migrations for this architecture.

A migration tool may still track and execute versioned `.sql` files.

### Deliberately use PostgreSQL

Prefer PostgreSQL for data-centric behavior:

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

Prefer atomic SQL operations over Python read-modify-write when PostgreSQL can enforce
the operation safely.

Example:

```sql
UPDATE accounts
SET balance = balance - %(amount)s
WHERE id = %(id)s
  AND balance >= %(amount)s
RETURNING *;
```

Do not load relational data into Python merely to reproduce joins, filtering,
aggregation, locking, or integrity rules that PostgreSQL handles naturally.
---

## `gateway/` — External Systems

`gateway/` contains concrete adapters for external systems:

```text
gateway/payment.py
gateway/email.py
gateway/storage.py
gateway/openai.py
```

Examples:

- payment providers;
- email/SMS providers;
- OpenAI or other AI services;
- MinIO/S3/object storage;
- third-party HTTP APIs.

Pydantic is mandatory for external provider request/response payloads.

If a workflow genuinely benefits from an abstract contract, define the Protocol in
the relevant domain module and implement it in the gateway module.

Do not create a separate `ports/` directory by default.

Concrete gateway objects are instantiated/wired in composition-root code such as
`main.py`, `state.py`, or `api/deps.py`, not in `domain/`.

---

## Exceptions and Boundary Failure Conversion

`try/except` is **not** business control flow.

Rules:

- no `try/except` in domain;
- no `try/except` in workflow business logic;
- no routine `try/except` in API business handling;
- catch third-party/library exceptions at genuine I/O boundaries such as DB and
  gateway implementations;
- convert expected external exceptions immediately to typed `Failure(...)` values;
- helpers/pipelines should be total over their declared business inputs;
- expected failures are values, not raises.

A global framework exception handler may exist only as a last-resort safety net for
genuine programmer/runtime faults. It must not replace typed expected failures.

In production, a genuine unexpected fault should become a safe 500 response without
uncontrolled traceback output.

---

## Logging and Observability

Use one reusable **higher-order logging decorator** for executable business
operations.

Apply it to business/API/workflow/gateway/database operations that participate in
application behavior. Logging implementation functions and the low-level DB log sink
are exempt to prevent recursive logging.

Use `loguru` unless the project already has an explicitly chosen structured logger.

Every decorated business operation must produce:

1. structured JSON on stdout;
2. a structured row in PostgreSQL `logs`.

The decorator must preserve the original business container. Logging failure must
**never** replace the original `Success`/`Failure` or turn a business `Success` into a
failure.

Recommended log columns:

```text
log_id
created_at
correlation_id
file_name
function_name
input_args
output
status          # success | note | fail | warn
duration_ms
error_type
```

### Correlation context

Attach request/workflow correlation context consistently. Prefer context-local/bound
logging context rather than manually threading a correlation ID through every
business function when the logging library/runtime can carry it safely.

### Redaction

Never log raw:

- passwords;
- access/refresh tokens;
- API keys;
- private credentials;
- secrets;
- sensitive PII unless explicitly sanitized and required.

Sanitize both stdout and DB-persisted logs.

### Tracebacks

Expected failures are typed `Failure` values, not traceback-driven control flow.

Production defaults:

```text
LOG_LEVEL=INFO
LOG_TRACEBACKS=false
```

Development/debug may enable:

```text
LOG_LEVEL=DEBUG
LOG_TRACEBACKS=true
```

When enabled, traceback information is diagnostic metadata only. Expected failures
still return typed `Failure` values and callers still handle them normally.

Do not use `logger.exception()` or automatic traceback emission for routine expected
failures. Boundary wrappers may include exception details in debug mode when mapping
an external exception to a typed failure.

---

## Formatting and Code Shape

Prefer functions over classes for behavior.

Keep functions small enough to understand in one pass when practical.

Write fluent chains vertically:

```python
result
.map(step_one)
.bind(step_two)
.lash(recover)
```

For non-trivial multiline calls, put arguments one per line with trailing commas:

```python
func(
    arg1,
    arg2,
    arg3,
)
```

Prefer readable named intermediate functions/values over dense clever expressions.

Prefer immutable transformations over hidden mutation.

Avoid boolean flag arguments when separate functions or richer typed values describe
intent better.

Keep imports organized and remove dead code while touching a module.

Match existing public interfaces unless the task explicitly requires changing them.

---

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
Python query layer
repository-style Unit of Work
mandatory workflow/application hop
ports directory
Alembic Python migrations
service classes for workflow logic
attrs-based domain models
```

Introduce an abstraction only when a concrete project requirement justifies it.

---

## Testing Rules

Use `pytest` and test observable typed behavior.

When relevant, cover:

- `Success` branches;
- `Failure` branches;
- `Some` branches;
- `Nothing` branches;
- each meaningful `singledispatch` registration;
- the default singledispatch unsupported-type failure;
- workflow composition with real domain dataclasses/enums;
- API Result-to-HTTP mapping;
- database/gateway exception-to-typed-Failure mapping;
- Pydantic boundary validation;
- logging redaction;
- correlation context;
- logging result preservation.

Assert on structured typed values and fields, not only message text.

Test shape:

- focused unit tests for computational helpers and workflow pipelines;
- integration tests for SQL/PostgreSQL behavior and gateways;
- e2e tests for public HTTP behavior when needed;
- keep fixtures small/local until reuse clearly justifies shared fixtures;
- do not mock immutable domain dataclasses/enums unnecessarily;
- expected failures should be asserted as `Failure(ErrorType(...))`, not expected
  raised exceptions.

---

## Delivery Quality Gate

Before finishing a change:

1. Read the relevant implementation, tests, and call sites.
2. Make the smallest safe change that satisfies the task.
3. Add/update focused `pytest` tests for behavior changes.
4. Run the narrowest relevant tests first.
5. Expand the test scope when risk warrants it.
6. Run `ruff`.
7. Run `ty`.
8. Run `basedpyright .`.
9. Resolve basedpyright errors; warnings may remain.
10. If `ty` and basedpyright disagree, prefer a `ty`-compatible design while keeping
    basedpyright errors resolved.
11. Prefer structural fixes over casts, ignores, or suppressions.
12. If suppression is unavoidable, keep it narrow and explain it when appropriate.
13. State clearly if a required check could not be run or still fails.

Treat linting/static analysis as delivery gates, not optional polish.

---

## Structural Rules for Agents

Apply these rules in order:

1. Start flat.
2. Keep API modules flat by capability until they become too large.
3. Route simple CRUD/query/report operations directly from API to SQL.
4. Create a workflow only for real orchestration.
5. Keep workflow modules action-named and flat by default.
6. Workflow modules contain functions only.
7. Helpers start with `_`, remain module-private, and return `Result` or `Maybe`.
8. Pipeline success types are domain dataclasses/enums, never primitives.
9. Keep domain modules flat and framework-free.
10. Domain class forms are Enum, frozen slotted dataclass, and Protocol only.
11. Domain contains no executable functions.
12. Put expected typed failure values in `error.py`.
13. Use Pydantic at every external I/O boundary.
14. Do not create persistence-interface Pydantic models merely to mirror SQL rows.
15. Keep all runtime SQL in `.sql` files.
16. Keep schema history in SQL migrations.
17. Prefer PostgreSQL constraints and atomic operations for integrity/concurrency.
18. Prefer joins/aggregation/CTEs/window functions in PostgreSQL over Python-side
    relational processing.
19. Keep `db/` generic and feature-agnostic.
20. Keep concrete external providers in `gateway/`.
21. Define a domain Protocol only when a real workflow needs an abstract contract.
22. Prefer `singledispatch` for behavior varying by first domain argument runtime type.
23. Prefer fluent `returns` composition over manual container unwrapping.
24. Convert DB/gateway exceptions into typed failures at the boundary.
25. Decorate executable business operations with the shared logging decorator, except
    logger internals/sinks.
26. Keep logs structured JSON, correlated, redacted, and persisted to PostgreSQL.
27. Add hierarchy only when real complexity makes flat modules hard to maintain.
28. Never invent repository/UoW/mapper/port layers for hypothetical replaceability.

---

## Agent Decision Checklist

Before writing code, classify the work:

```text
External I/O?
    -> Pydantic boundary model

Simple database operation?
    -> API -> db/sql -> .sql -> PostgreSQL

Non-trivial business orchestration?
    -> workflow pipeline

Pure computational workflow step?
    -> private _helper

Business data/state?
    -> domain Enum / frozen dataclass

Abstract external behavior genuinely needed?
    -> domain Protocol

Expected failure?
    -> typed immutable value in error.py + Failure(...)

Behavior varies by first domain argument runtime type?
    -> functools.singledispatch

External exception?
    -> catch at DB/gateway boundary and convert to Failure

Relational/data logic?
    -> SQL/PostgreSQL

Generic PostgreSQL lifecycle/execution machinery?
    -> db/

Concrete provider implementation?
    -> gateway/
```

If none of these rules gives a new abstraction a concrete responsibility, do not add
it.

---

## Quick Quality Checklist

Before declaring work complete, verify:

```text
Structure
[ ] Simple DB operations bypass workflow.
[ ] Non-trivial orchestration lives in workflow.
[ ] No ORM/repository/mapper/query-layer ceremony was added without need.
[ ] Modules remain flat unless complexity requires otherwise.

Modeling
[ ] External I/O is validated by Pydantic.
[ ] Domain contains only Enum/dataclass/Protocol classes.
[ ] Domain contains no executable functions.
[ ] Workflow contains no classes.
[ ] Expected failures are immutable typed values in error.py.
[ ] No new Any-shaped typing was introduced.

Functions
[ ] Every function is fully annotated.
[ ] Helpers start with _ and are module-private.
[ ] Pipeline successes are non-primitive domain values.
[ ] Expected outcomes are wrapped in Result/Maybe.
[ ] Fluent returns chaining is preferred over manual unwrapping.
[ ] singledispatch is used where type-driven behavior genuinely benefits from it.

Errors
[ ] Expected failures are Failure(ErrorValue(...)).
[ ] try/except exists only at genuine I/O/runtime safety boundaries.
[ ] Boundary exceptions are converted immediately to typed failures.

Logging
[ ] Business operations use the shared logging decorator.
[ ] Logger internals/sinks are excluded from decoration.
[ ] Stdout logs are JSON.
[ ] Logs persist to PostgreSQL.
[ ] Correlation context is present.
[ ] Secrets/PII are redacted.
[ ] Logging failure does not change the business Result.
[ ] Production tracebacks are off by default.

Verification
[ ] Success/Failure tested where relevant.
[ ] Some/Nothing tested where relevant.
[ ] singledispatch variants/default tested where used.
[ ] pytest run.
[ ] ruff run.
[ ] ty run.
[ ] basedpyright errors resolved.
[ ] Any unavailable/failing checks reported clearly.
```