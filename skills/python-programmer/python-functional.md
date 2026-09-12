---
name: python-functional

description: >
  Functional Python rules for typed workflows using returns.Result/Maybe,
  private helpers, business pipelines, singledispatch, and iterator-based
  transformations.
---

# Python Functional Programming

## Functional First

- Prefer functions over classes.
- `workflow/` contains functions only; never define classes there.
- Keep business data in domain enums/dataclasses and contracts in Protocols.
- Keep side effects at explicit DB/gateway/API boundaries.
- Prefer immutable transformations over mutation.
- Every executable business function has explicit parameter and return annotations.

## Result and Maybe

Use `returns` as the default expected-control-flow model:

```text
Result[T, E] -> Success(T) | Failure(E)
Maybe[T]     -> Some(T) | Nothing
```

All business functions return `Result[T, E]`, except private computational helpers
that may return `Maybe[T]` when absence is the only alternative.

Prefer fluent operations such as `.map()`, `.bind()`, `.lash()`, `.alt()`, and
`.value_or()` over repeatedly matching/unwrapping containers.

Use `match` only when the business flow genuinely branches over a finite domain
state, especially enum values.

## Workflow Function Kinds

### Helpers

Private helper functions:

- start with `_`;
- are never imported from other modules;
- are computational rather than business workflows;
- may accept/return primitive values;
- return `Result[T, E]` or `Maybe[T]`.

### Pipelines

Public pipeline functions:

- represent business actions;
- orchestrate helpers and possibly other pipelines;
- accept domain-typed business values;
- return `Result[DomainType, ErrorType]`;
- never use a primitive success type;
- compose work with fluent Result/Maybe operations whenever practical.

Keep pipeline code linear and readable.

## Type-Driven Function Dispatch

Use `functools.singledispatch` when behavior varies by the runtime class of the first
domain argument.

Prefer this to long `isinstance` chains.

The default implementation returns a typed `Failure`, not
`NotImplementedError`/an expected exception.

```python
@singledispatch
def process(
    value: object,
) -> Result[ProcessedValue, AppError]:
    return Failure(
        UnsupportedDomainType(type_name=type(value).__name__),
    )


@process.register
def _(
    value: DomainA,
) -> Result[ProcessedValue, AppError]:
    ...
```

`singledispatch` dispatches only on the first argument's runtime class. It does not
dispatch on different members of the same Enum; use explicit enum branching there.

## Iterator-Based Collection Work

Avoid imperative `for` loops when the same operation is clearer as a declarative
transformation.

Prefer:

- `map`;
- `filter`;
- generators/comprehensions where clearer;
- `itertools` for lazy composition;
- `returns.iterables.Fold` for collections containing Result/Maybe values.

Use `.iter().map().filter()` fluent syntax only when the concrete iterable library in
the project actually supports it. Stdlib `itertools` does not provide that method
chain API.

## Exceptions

Do not use `try/except` for business control flow.

Catch external/library exceptions only in DB/gateway boundary implementations and
convert expected failures immediately into typed `Failure` values.

Pure helpers and pipeline functions should be total over their declared business
inputs: expected error cases are values, not raises.

## Formatting

Write fluent chains vertically:

```python
result
.map(step_one)
.bind(step_two)
.lash(recover)
```

For non-trivial multiline calls, place arguments one per line with trailing commas.
