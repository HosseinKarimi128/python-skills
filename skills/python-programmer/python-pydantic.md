---
name: python-pydantic

description: >
  Pydantic rules for external I/O boundaries. API, gateway, configuration, and
  other external payloads must be validated; domain/workflow models must not use
  Pydantic.
---

# Pydantic and Boundary Typing

## Mandatory External I/O Validation

Use Pydantic for every external I/O boundary:

- FastAPI request/response bodies;
- path/query parameter models when modeled explicitly;
- gateway provider request/response payloads;
- configuration and environment input;
- queue/event payloads;
- file/CLI/external payloads when present.

Validate external data immediately when it crosses the boundary.

Use `Field`, validators, discriminated unions, constrained types, and exact enums as
needed to make invalid boundary data unrepresentable.

## Do Not Use Pydantic Internally by Default

Pydantic does not belong in:

```text
domain/
workflow/
```

Domain business values are frozen slotted dataclasses and enums. Protocols describe
contracts when genuinely needed.

Do not create database-interface Pydantic models simply to mirror SQL rows or ORM
entities. This architecture has no ORM persistence model layer.

For simple CRUD:

```text
SQL row -> API Pydantic response model
```

For a business workflow:

```text
SQL row -> domain dataclass (when the workflow needs a domain value)
```

Gateway implementations may use Pydantic models internally for external provider
payloads, then convert validated data into domain values required by a workflow.

## Strict Typing

- Avoid `Any`, `dict[str, Any]`, and `list[Any]`.
- Prefer precise fields, typed aliases, Protocols, `object`, and exact nested models.
- Never let weak third-party payload typing leak past the boundary; validate/normalize
  it immediately.
- Keep nullable fields explicit with `T | None` only when null is valid external data.
- Do not use raw `None` as a business failure signal; use `Maybe`/`Result` after the
  boundary.

## API Result Contract

API business functions use Pydantic at the HTTP boundary while keeping failures in
`Result`:

```python
Result[UserResponse, AppError]
```

A centralized HTTP adapter converts the Result into the actual FastAPI response or
mapped HTTP error. Do not duplicate result-unwrapping logic in each route.
