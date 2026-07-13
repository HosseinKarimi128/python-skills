---
name: python-programmer

description: >
  Production-grade Python skill for type-safe, validation-heavy,
  test-driven code. Prefer attrs, Pydantic, pytest, ruff, ty,
  and returns-style functional pipelines.
---

# Python Programmer

Use this skill for Python work that must be clean, typed, tested, and production-ready.

## Important Notes

- Use the `returns` package for explicit success, failure, and optional values.
- Avoid OOP principle. Put functional programming approach first.
- Prefer functions over classes. If a class is necessary, you must use `attrs` and avoid using `__init__` method.
- If there is a design decision, a complex problem, or task ambiguity, discuss it with the user first before taking action.
- Write type-safe code using Pydantic models for validated I/O boundaries and `returns` utilities for functional control flow.
- Write code in a pipe-like vertical style:
  - each chained `.` call goes on its own line,
  - when the function argument are more then 3, each argument goes on its own line in multiline calls.
- Use explicit typed models and errors instead of loose primitives or untyped dicts.
- Avoid `Any` in all code you write or modify. Treat `Any` as a last-resort escape hatch that requires strong justification, and prefer exact domain types, `Protocol`s, recursive JSON aliases, `object`, `TypeAdapter`, or other concrete typed boundaries instead.
- Log everythings! Log using `loguru` lib.

## Type Discipline — Three Kinds of Domain Types

Every value that crosses a function boundary must have an explicit type. There are three categories, each with a dedicated home:

### 1. I/O Contracts — Pydantic `BaseModel` (validation at the edge)

**Only** HTTP request bodies, response bodies, path/query params, and external API payloads use Pydantic. These live in `domain/models/` and are suffixed with `Request`, `Response`, `PathRequest`, `QueryRequest`, etc.

```python
class CreateUserRequest(BaseModel):
    """POST /api/v1/users"""
    model_config: ClassVar[ConfigDict] = ConfigDict(extra="forbid")
    name: str = Field(min_length=1)
    mobile: str = Field(min_length=1)
```

### 2. Internal Communication Types — Plain Python (zero validation overhead)

Types that pass between domain services, repositories, and protocols must **NOT** subclass Pydantic. They are pure data carriers and must be plain `@dataclass(frozen=True, slots=True)` instances, `NamedTuple`, or simple `type` aliases. These also live in `domain/models/` but are **not** I/O models — they are internal value objects.

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class RouteIntentDecision:
    """Structured decision returned by intent classification."""
    decision: str = "clarify"
    query: str | None = None
    fetch_hint: str | None = None
    reason: str | None = None
    conversation_title: str | None = None

@dataclass(frozen=True, slots=True)
class EbornixExecuteResult:
    """Structured response from ebornix query execution."""
    duckdb_path: str
    duckdb_table: str
    row_count: int
```

Why plain types for internals?
- **No runtime validation overhead** — internal code already trusts its own types.
- **Immutable by default** — `frozen=True` gives Rust-like semantics.
- **Clean pattern matching** — `match` on `dataclass` instances is exhaustive and readable.
- **Explicit nullability** — use `returns.Maybe` instead of `None` when absence is meaningful.

### 3. Error Variants — Typed exceptions in `domain/errors/`

Every failure mode is a named type inheriting from `DomainError`. These are the `E` in `Result[T, E]`.

```python
class DomainError(Exception):
    def __init__(self, message: str) -> None:
        self.message = message
        super().__init__(message)

class EbornixError(DomainError): ...
class LLMGatewayError(DomainError): ...
class UserNotFoundError(DomainError): ...
```

### 4. Enums — Exhaustive tagged unions in `domain/enums/`

Every discriminated union or state machine uses an `enum.Enum` or `enum.StrEnum`. These enable exhaustive `match-case` branching, eliminating open-ended `if-elif` chains.

```python
from enum import StrEnum

class IntentDecision(StrEnum):
    CLARIFY = "clarify"
    FETCH = "fetch"
    ANALYTICAL = "analytical"
```

## Required Workflow

- Read the relevant code, tests, and call sites before editing.
- Make the smallest safe change that solves the task.
- Add or update `pytest` tests for behavior changes.
- Run the tests you add or change.
- Run `basedpyright .`, `ruff`, and `ty`, and finish only when they pass.
- For `basedpyright`, fix errors only and skip warnings.
- If `ty` and `basedpyright` conflict, prefer `ty`-compatible fixes while keeping `basedpyright` errors resolved.
- If a check cannot be run or still fails, say so clearly.

## Coding Rules

- Prefer small pure functions and immutable-style transformations.
- Keep side effects at boundaries.
- Use protocols to generalize the mutual behaviors. Use Protocol subclassing as we use trait in Rust.
- Use enums as we do by subclassing enum. Pydantic BaseModel is reserved for I/O contracts ONLY.
- You should define traits in `/protocols`, so functions in `/services` could implement them. (python duck typing)
- All functions should have typing hints. Function inputs and outputs that cross service boundaries must be plain `@dataclass(frozen=True, slots=True)` value objects or enums unless they are validated I/O models.
- The above points are true for function arguments.
- Use `returns` utilities for type definitions. It helps you to define type models, as we define `Struct` in Rust.
- Validate I/O immediately with Pydantic.
- You MUST use typed contracts over bare primitives when values have domain meaning. `returns` would help you with it.
- Do not introduce `typing.Any` or `dict[str, Any]` / `list[Any]` style annotations unless the user explicitly asks for a looser boundary or no exact type is realistically expressible. If a third-party API leaks weak typing, contain it at the boundary and convert it immediately into an exact typed model.
- Match existing public interfaces unless the task requires changing them.
- Avoid unnecessary abstraction and inheritance.
- Prefer operation chaining over case branching when the flow can stay linear.
- **No `try`/`except` in business logic.** Convert all potentially-raising operations into `returns.Result[T, E]` via `safe` / `future_safe` wrappers and model failure with typed error types.
- **Prefer pattern matching over `if-elif` only when branching is unavoidable.** Use `match-case` on `Result`, `Maybe`, and enums for exhaustive branching.

## Pattern Matching Over Result and Maybe

Instead of chaining `.is_success()`, `.is_some()`, or `.unwrap()`, use `match` for exhaustive, readable branching:

```python
def handle_result(result: Result[User, DomainError]) -> UserResponse:
    match result:
        case Success(user):
            return UserResponse.from_domain(user)
        case Failure(UserNotFoundError() as err):
            raise HTTPException(status_code=404, detail=err.message)
        case Failure(PermissionDeniedError() as err):
            raise HTTPException(status_code=403, detail=err.message)
        case Failure(err):
            raise HTTPException(status_code=500, detail=err.message)
```

For `Maybe`:

```python
def find_user(users: Vec[User], user_id: int) -> Maybe[User]:
    return users.iter().find(lambda u: u.id == user_id)

match find_user(all_users, 42):
    case Some(user):
        return user
    case Nothing:
        raise HTTPException(status_code=404, detail="User not found")
```

For enums:

```python
match intent.decision:
    case IntentDecision.CLARIFY:
        return clarify_response(prompt)
    case IntentDecision.FETCH:
        return fetch_response(prompt)
    case IntentDecision.ANALYTICAL:
        return analytical_response(prompt)
```

Use enums as the discriminant for pattern matching whenever the branch set is finite and meaningful.

## Prefer `functools.singledispatch` for Type-Based Business Logic

When business logic varies by the concrete type of a domain object, prefer `functools.singledispatch` over `if/elif isinstance(...)` or `match/case`.

This is especially useful for processing:

- Commands
- Events
- Requests
- Domain models
- DTOs
- Policies
- Strategy objects
- Typed AST nodes
- Validation results

### Example

```python
from functools import singledispatch
from dataclasses import dataclass


class PaymentMethod:
    """Marker base class."""


@dataclass(frozen=True)
class CreditCard(PaymentMethod):
    token: str


@dataclass(frozen=True)
class BankTransfer(PaymentMethod):
    iban: str


@dataclass(frozen=True)
class CryptoWallet(PaymentMethod):
    address: str


@singledispatch
def process_payment(method: PaymentMethod) -> PaymentResult:
    raise TypeError(f"Unsupported payment method: {type(method).__name__}")


@process_payment.register
def _(method: CreditCard) -> PaymentResult:
    ...


@process_payment.register
def _(method: BankTransfer) -> PaymentResult:
    ...


@process_payment.register
def _(method: CryptoWallet) -> PaymentResult:
    ...
```

Instead of

```python
def process_payment(method: PaymentMethod) -> PaymentResult:
    if isinstance(method, CreditCard):
        ...
    elif isinstance(method, BankTransfer):
        ...
    elif isinstance(method, CryptoWallet):
        ...
    else:
        ...
```

### Why

For domain-driven applications, each business type encapsulates a distinct concept. Registering one implementation per type:

- Keeps each business rule isolated.
- Makes adding new business types non-invasive.
- Eliminates growing `isinstance()` chains.
- Produces smaller, more testable functions.
- Follows the Open/Closed Principle.
- Leverages Python's runtime type hierarchy instead of manual dispatch.

### Guidelines

Prefer `singledispatch` when:

- Behavior is selected solely by the runtime type of the first argument.
- Each domain type represents a distinct business concept.
- New business types are expected over time.
- Each implementation is substantial enough to justify its own function.

Avoid it when dispatch depends on:

- Field values.
- Multiple arguments.
- Arbitrary predicates.
- State or configuration.

Use `match/case` for structural or value pattern matching, and `if/elif` for simple conditional logic.

### Architecture

Favor modeling business decisions as distinct types rather than flags or enums.

Prefer:

```python
ApproveOrder(...)
RejectOrder(...)
CancelOrder(...)
```

over

```python
OrderAction(action="approve")
OrderAction(action="reject")
OrderAction(action="cancel")
```

Well-modeled domain types naturally enable `singledispatch`, resulting in code that is easier to extend, test, and reason about.

## Testing Rules

- Use `pytest`.
- Test success and failure paths.
- Assert on structured typed values, not just message text.
- Prefer focused unit tests first, then broader tests if needed.

## Quality Gate

- Code must pass `basedpyright .`.
- Code must pass `ruff`.
- Code must pass `ty`.
- `basedpyright` warnings are allowed; prioritize resolving `basedpyright` errors.
- When `ty` and `basedpyright` disagree, `ty` is the source of truth.
- Keep formatting readable and diff-friendly.
- Prefer structural fixes over ignores or suppressions.
