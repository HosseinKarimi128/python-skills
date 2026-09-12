# Python Quality Checklist

Use this as the short implementation checklist for `python-programmer`.

## Structure

- Follow the PostgreSQL-first `python-project-structure` skill.
- Simple CRUD/query/report: `api -> SQL`.
- Non-trivial business work: `api -> workflow -> domain / SQL / gateway`.
- Keep modules flat until complexity requires grouping.
- No ORM/repository/mapper/query-layer ceremony by default.

## Modeling

- External I/O uses Pydantic.
- Domain contains only data-only `Enum`, frozen slotted `dataclass`, and `Protocol` classes.
- No executable functions in domain modules.
- No classes in workflow modules.
- Expected errors are typed immutable values in upper-level `error.py`.
- Avoid `Any` and weak untyped dictionaries.

## Functions

- Every executable business function is fully annotated.
- Helpers start with `_`, remain module-private, and return `Result` or `Maybe`.
- Pipeline success values are domain dataclasses/enums, never primitives.
- Prefer fluent `returns` chaining over manual unwrapping.
- Prefer `singledispatch` for behavior varying by first domain argument type.
- Prefer iterator transformations to imperative loops when clearer.

## Errors

- Expected failures use `Failure(ErrorValue(...))`.
- `try/except` belongs only at genuine DB/gateway I/O boundaries.
- Boundary exceptions are converted immediately to typed failures.
- A global exception handler is last-resort only.

## Logging

- Decorate every executable business operation with the shared logging decorator.
- Exclude logger internals and the low-level DB log sink.
- Emit JSON stdout and persist to the `logs` table.
- Preserve the original business Result even if logging persistence fails.
- Include correlation context and redact secrets/PII.
- Production tracebacks off; debug tracebacks may be enabled by environment.

## Verification

- Test `Success` / `Failure`.
- Test `Some` / `Nothing`.
- Test singledispatch variants/default failure where used.
- Run focused `pytest`.
- Run `ruff`.
- Run `ty`.
- Run `basedpyright .` and resolve errors.
- Report any check that could not run.
