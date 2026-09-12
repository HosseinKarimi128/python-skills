---
name: python-programmer

description: >
  Functional, strongly typed Python rules for PostgreSQL-first FastAPI backends.
  Use Pydantic at external I/O boundaries, domain dataclasses/enums/protocols for
  business types, returns.Result/Maybe for explicit control flow, and functions-only
  workflows with strict quality and observability rules.
---

# Python Programmer

Use these rules for production Python code. Keep the design functional, explicit,
typed, small, and aligned with the `python-project-structure` skill.

## Architecture in One Minute

```text
api/       HTTP boundary; Pydantic request/response models
workflow/  non-trivial business orchestration; functions only
domain/    data-only Enum, frozen dataclass, and Protocol classes
db/        generic PostgreSQL execution/lifecycle machinery
gateway/   concrete external-system adapters; Pydantic at external I/O
error.py   typed application failure values
sql/       all runtime SQL and SQL migrations
```

Normal paths:

```text
simple CRUD/query/report:
api -> db/sql -> .sql -> PostgreSQL

business workflow:
api -> workflow -> domain / db/sql / gateway
```

Do not reintroduce ORM, repository, mapper, query-layer, mandatory workflow, or
repository-style Unit of Work ceremony unless a concrete requirement justifies it.

## Type Discipline

- Every function and method must have explicit parameter and `->` return annotations.
- Avoid `Any`, `dict[str, Any]`, `list[Any]`, implicit optionals, and weakly typed
  boundaries. Use exact types, `object`, Protocols, aliases, or typed containers.
- Prefer immutable values. Domain dataclasses use `@dataclass(frozen=True, slots=True)`.
- Use meaningful typed values instead of unstructured dictionaries when data has
  business meaning.
- Do not use `attrs` by default.

## Pydantic Is Mandatory at External I/O Boundaries

Use Pydantic for data entering or leaving the application through an external
boundary, including:

- FastAPI request, response, path, and query models;
- gateway provider request/response payloads;
- configuration/environment input;
- queue/event payloads, files, CLI input, or other external payloads when present.

Validate external data immediately at the boundary.

Do not put Pydantic models in `domain/` or `workflow/`.
Do not create database-specific Pydantic models merely to duplicate SQL result
shapes. SQL is executed directly from `.sql` files; direct CRUD endpoints may
validate returned rows into their API response model, and workflows may convert
rows into domain dataclasses when a business type is actually needed.

## Domain Rules

Keep `domain/` flat by default. A domain module may define only data/contract
classes of these kinds:

1. `Enum` / `StrEnum`
2. `@dataclass(frozen=True, slots=True)`
3. `Protocol`

Domain modules contain no executable functions. Dataclasses are data-only: do not
add business methods, helper methods, properties, or I/O behavior. Protocol methods
contain signatures only.

Define a Protocol only when a workflow genuinely needs an abstract contract. Most
ordinary web-backend operations need no Protocol. Never create repository Protocols
just to hide PostgreSQL.

## Error Values

Expected failures are values, not exception-based business control flow.
Define application failure types in the upper-level `error.py`, normally as frozen
slotted dataclasses or enums. Do not require them to inherit from `Exception`.

```python
@dataclass(frozen=True, slots=True)
class UserNotFound:
    user_id: UUID


@dataclass(frozen=True, slots=True)
class PaymentUnavailable:
    provider: str
```

Every expected failure that callers may need to handle must have an explicit typed
variant.

## `returns` Is the Default Control-Flow Model

Use `returns` for expected success, failure, and absence.

```text
Result[T, E] -> Success(T) | Failure(E)
Maybe[T]     -> Some(T) | Nothing
```

All executable business functions return a `Result[T, E]`, except private workflow
helpers that may return `Maybe[T]` when absence is the only alternative.

Never return a bare business success value from a pipeline/API/gateway operation.
Never use `None` as an implicit failure signal when `Maybe` expresses the contract.

Prefer fluent container operations such as:

```text
.map(...)
.bind(...)
.lash(...)
.alt(...)
.value_or(...)
```

Prefer chained `.` composition over match-first or nested conditional handling when
the flow can remain linear. Use `match` when there is a genuine finite business
branch, especially enum values; do not use it merely to unwrap every Result.

## Workflow Rules

`workflow/` contains functions only. Never define classes there.

There are two workflow function kinds.

### Helper functions

- Name starts with `_`.
- Private to the module; never import it from another file.
- Performs computational/transformation work, not a business workflow.
- May accept and return primitives.
- Returns `Result[T, E]` or `Maybe[T]`.
- A helper success type may be primitive.

```python
def _calculate_total(
    subtotal: Decimal,
    tax: Decimal,
) -> Result[Decimal, AppError]:
    ...
```

### Pipeline functions

- Public functions representing business work.
- Orchestrate helpers and, when useful, other pipeline functions.
- Accept business-typed domain values when operating inside the business layer.
- Return `Result[SuccessType, FailureType]`.
- `SuccessType` must not be a primitive. It must be a domain dataclass or enum.
- Compose work primarily through `Result`/`Maybe` chaining.

```python
def checkout_order(
    order: Order,
    payment: PaymentProvider,
) -> Result[CheckoutResult, AppError]:
    ...
```

Pipeline functions should describe business actions, not simple CRUD wrappers.

## API Function Rules

API functions follow the same Result discipline, but their boundary types are
Pydantic models rather than domain dataclasses.

A route operation should normally return conceptually:

```python
Result[ResponseModel, AppError]
```

Use one centralized HTTP Result adapter/decorator to translate:

```text
Success(ResponseModel) -> normal FastAPI response
Failure(NotFound)      -> mapped 404
Failure(InvalidInput)  -> mapped 400/422
Failure(DependencyDown)-> mapped 503
Failure(Unexpected)    -> mapped 500
```

Do not duplicate Result-to-HTTP `match`/`if` logic in every route.
Simple CRUD/query/report routes may execute SQL directly through `db/sql` without a
workflow module.

## Gateway and Database Boundaries

`try/except` is not business control flow.

- Do not write `try/except` in workflow/domain/API business logic.
- Catch external/library exceptions only at genuine I/O boundaries such as
  database and gateway implementations.
- Convert expected boundary exceptions immediately into typed `Failure(...)` values.
- Keep provider payloads validated by Pydantic inside gateways.
- Keep SQL in `.sql` files; Python only loads/executes the statement and handles the
  typed result/failure boundary.

A global framework exception handler may exist only as a last-resort safety net for
unexpected programmer/runtime failures. It must not replace typed expected failures.

## Type-Driven Dispatch with `functools.singledispatch`

Prefer `@singledispatch` when behavior naturally varies by the runtime type of the
first domain argument. This is the preferred Python approximation of Elixir-style
function clauses for type-based business behavior.

```python
@singledispatch
def calculate_price(
    item: object,
) -> Result[Price, AppError]:
    return Failure(
        UnsupportedDomainType(type_name=type(item).__name__),
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

- Prefer `singledispatch` over `isinstance` chains when dispatch is truly type-based.
- The default implementation returns a typed `Failure`; do not raise
  `NotImplementedError` for an expected unsupported type.
- Dispatch is on the first argument's runtime class only.
- Do not use `singledispatch` to distinguish members of the same Enum; use explicit
  enum branching for that.
- Registered implementations remain functions; do not introduce service classes.

## Functional Collection Processing

Avoid imperative `for` loops when a transformation can be expressed clearly with
iterator operations.

Prefer:

- `map` and `filter`;
- generator/comprehension forms when clearer;
- `itertools` for lazy composition;
- `returns.iterables.Fold` when composing collections of `Result`/`Maybe` values.

Use fluent `.iter().map().filter()` style only when the concrete iterable abstraction
in the project actually provides that API; Python stdlib `itertools` itself does not.

## Logging and Observability

Define one reusable higher-order logging decorator and apply it to every executable
business function/operation. Logging implementation functions and the low-level DB
log sink are exempt to prevent recursive logging.

The decorator must:

- emit structured JSON to stdout;
- persist a structured record to the PostgreSQL `logs` table;
- record function/file identity, sanitized inputs, output/failure, status, duration,
  and correlation context;
- preserve the original `Success`/`Failure`; logging failure must not change the
  business result;
- redact secrets, tokens, passwords, credentials, and sensitive payload fields.

Recommended log fields:

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

Production stdout remains JSON and should not include tracebacks by default.
Control diagnostic verbosity with environment configuration such as:

```text
LOG_LEVEL=INFO
LOG_TRACEBACKS=false
```

Development/debug may enable:

```text
LOG_LEVEL=DEBUG
LOG_TRACEBACKS=true
```

Tracebacks are diagnostic detail only. Even in debug mode, expected failures remain
`Failure` values and callers handle them normally.

## Formatting

- Keep fluent chains vertical: one chained `.` call per line.
- For non-trivial multiline calls, put arguments one per line with trailing commas.
- Prefer small, named functions over deeply nested expressions.
- Keep modules flat until actual complexity requires grouping.

## Testing

Use `pytest`.

Test:

- `Success` and `Failure` paths;
- `Some` and `Nothing` paths;
- every meaningful `singledispatch` variant and its default failure;
- HTTP Result-to-response/error mapping;
- gateway/database exception-to-Failure mapping;
- business pipeline behavior using typed domain values.

Assert on typed structured values, not only error-message text.

## Delivery Quality Gate

Before finishing a change:

1. Read the relevant implementation, tests, and call sites.
2. Make the smallest safe change that satisfies the task.
3. Add/update focused `pytest` tests for behavior changes.
4. Run the relevant tests.
5. Run `ruff`.
6. Run `ty`.
7. Run `basedpyright .`; resolve errors, warnings may remain.
8. If `ty` and basedpyright disagree, prefer a `ty`-compatible solution while keeping
   basedpyright errors resolved.
9. Prefer structural fixes over casts, ignores, or suppressions.
10. State clearly if a required check could not be run or still fails.

## Agent Checklist

Before writing code, determine:

```text
External I/O?
    -> Pydantic boundary model

Simple database operation?
    -> API -> SQL directly

Non-trivial business orchestration?
    -> workflow pipeline function

Pure computational step inside workflow?
    -> private _helper function

Business data/contract type?
    -> domain Enum / frozen dataclass / Protocol

Expected failure?
    -> typed value in error.py + Failure(...)

Behavior varies by first domain argument type?
    -> singledispatch

External exception?
    -> catch only in DB/gateway boundary and convert to Failure
```

Do not create abstractions, classes, models, or layers unless these rules give them a
real responsibility.
