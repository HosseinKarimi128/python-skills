---
name: python-testing

description: >
  Pytest rules for Result/Maybe workflows, singledispatch behavior, HTTP adapters,
  and I/O-boundary failure conversion.
---

# Python Testing Workflow

Use `pytest` and keep tests focused on observable typed behavior.

## Required Coverage

When behavior changes, test the relevant cases:

- `Success` and `Failure` branches for Result-returning functions;
- `Some` and `Nothing` branches for Maybe-returning helpers;
- each meaningful `singledispatch` registration;
- the default singledispatch unsupported-type `Failure`;
- workflow pipeline composition with domain dataclasses/enums;
- API Result-to-HTTP response/error mapping;
- gateway/database external-exception-to-typed-Failure mapping;
- Pydantic boundary validation for external payloads;
- logging redaction/correlation/result preservation when logging behavior changes.

Assert on typed values and structured fields, not only message strings.

## Test Shape

- Prefer focused unit tests for private computational helpers and workflow pipelines.
- Prefer integration tests for SQL execution, PostgreSQL behavior, and gateways.
- Use e2e tests for public HTTP behavior when needed.
- Keep fixtures small/local until real reuse justifies shared fixtures.
- Do not mock domain dataclasses/enums unnecessarily; construct real immutable values.
- Test expected failures as `Failure(ErrorType(...))`, not raised exceptions.

## Verification Order

1. Run the narrowest affected pytest selection.
2. Expand to the relevant package/suite when risk warrants it.
3. Run `ruff`.
4. Run `ty`.
5. Run `basedpyright .` and resolve errors.
6. Report clearly if a required check could not be executed.
