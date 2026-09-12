---
name: python-quality

description: >
  Strict typing, quality gates, formatting, and structured observability rules for
  functional Python backends.
---

# Python Quality and Observability

## Type and Static-Analysis Discipline

- Every function/method has explicit parameter and return annotations.
- Avoid `Any`, weak dictionaries, implicit optional values, and ambiguous unions.
- Prefer exact types, `object`, Protocols, aliases, and typed containers.
- Prefer immutable values and structural fixes over casts/ignores.
- Keep imports clean and remove dead code while touching a module.

Delivery gates:

```text
pytest
ruff
ty
basedpyright .
```

Resolve basedpyright errors; warnings may remain. If `ty` and basedpyright disagree,
prefer a `ty`-compatible design while keeping basedpyright errors resolved.

## Formatting

Keep fluent chains vertical:

```python
result
.map(step_one)
.bind(step_two)
.lash(recover)
```

For non-trivial multiline calls, put arguments one per line with trailing commas.
Prefer readable intermediate names over dense expressions.

## Structured Logging Decorator

Use one reusable higher-order logging decorator for executable business operations.
Apply it to API/business handlers, workflow helpers/pipelines, and other executable
business boundary operations. Logging implementation functions and the low-level DB
log sink are exempt to prevent recursion.

Use `loguru` for application logging unless the project already has an explicitly
chosen structured logger.

Every decorated operation must emit:

1. JSON to stdout;
2. a structured row in PostgreSQL `logs`.

The logging decorator must preserve the original return container. A logging/storage
failure must not turn a business `Success` into `Failure` or replace an existing
business failure.

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

## Correlation and Redaction

Attach request/workflow correlation context consistently. Prefer context-local bound
values rather than manually passing correlation IDs through every function.

Never log:

- passwords;
- access/refresh tokens;
- API keys;
- private credentials;
- raw secrets;
- sensitive PII unless explicitly sanitized and required.

Sanitize inputs/outputs before writing either stdout or database logs.

## Traceback Policy

Expected failures are typed `Failure` values, not tracebacks.

Production defaults:

```text
LOG_LEVEL=INFO
LOG_TRACEBACKS=false
```

Development/debug may use:

```text
LOG_LEVEL=DEBUG
LOG_TRACEBACKS=true
```

When traceback output is enabled, it is diagnostic metadata only. Expected control
flow still returns typed `Failure` values.

Do not use `logger.exception()`/automatic traceback output for routine expected
failures. Boundary wrappers may include exception details in debug mode when they
convert an external exception into a typed failure.

## No Unhandled Expected Errors

Expected database/gateway errors must be caught at their I/O boundary and converted
into typed failures. Business code must not rely on uncaught exceptions.

A global framework exception handler may remain as a last-resort guard for genuine
programmer/runtime faults. In production it should return a safe 500 response and
avoid uncontrolled traceback output to stdout.
