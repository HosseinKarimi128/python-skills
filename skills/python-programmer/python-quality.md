---
name: python-quality

description: >
  Ruff, ty, and static analysis discipline for Python.
  Treat linting and static analysis as delivery gates, not optional polish.
  Prefer structural fixes over suppressions.
  Use loguru for all logging; write application logs fully and professionally.
---

# Python Quality and Tooling Discipline

## Ruff and ty Discipline

- Write code that is clean enough to pass `ruff` without style-only rewrites afterward.
- Keep imports organized, remove dead code, and avoid overly complex branches or unused intermediate variables.
- Prefer straightforward expressions and clear naming over clever compactness.
- Prefer vertical formatting for readability and stable diffs:
  - chained calls are written one `.` call per line,
  - function calls with multiple arguments are split so each argument appears on its own indented line with a trailing comma.
- Check that new code is compatible with `ty` expectations: no ambiguous return types, no unchecked `None` paths, and no mismatched container types.
- Treat linting and static analysis as delivery gates, not optional polish. If the project has `ruff`, `ty`, SonarLint, or SonarQube rules configured, aim to satisfy them in the generated code rather than leaving cleanup for later.
- Prefer structural fixes over escapes: tighten types, use constrained aliases or validators, simplify control flow, and remove dead branches before reaching for casts, ignores, or suppressions.
- If a suppression is truly required, keep it narrow, explain why in a brief comment when the file style allows it, and avoid introducing broad file-level disables.
- Before finishing, run the narrowest relevant `ruff` and `ty` checks on the changed files when those tools are available. If local Sonar linting is available, run it too; otherwise, proactively avoid common Sonar issues such as constant-return validators, duplicated branches, needless conditionals, and overly complex methods.

## Logging with loguru

- **Use `loguru` for all application logging.** Do not use the standard library `logging` module directly.
- Configure a single `loguru` logger in `main.py` or a dedicated `lib/logging_config.py` with structured, production-ready settings:
  - JSON serialization for production environments.
  - Rotation, retention, and compression for log files.
  - Consistent log levels (`TRACE`, `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`).
- **Write application logs fully and professionally.** Every significant operation, decision point, and boundary crossing must be logged with enough context to reconstruct the flow without reading source code.
  - Log at `INFO` or higher for: request lifecycle events, service entry/exit, repository operations, external API calls, and configuration loads.
  - Log at `DEBUG` for: detailed internal state, parsed payloads, mapper transformations, and query parameters.
  - Log at `WARNING` for: recoverable issues, retries, degraded states, and non-critical validation failures.
  - Log at `ERROR` for: unrecoverable domain failures, repository exceptions, and boundary errors that propagate as `returns.Failure`.
  - Log at `CRITICAL` for: startup failures, fatal misconfigurations, and unhandled exceptions that terminate the process.
- Include structured context in every log call: correlation IDs, user IDs, request paths, operation names, and relevant Pydantic model fields (sanitized of secrets).
- Never log sensitive data (passwords, tokens, PII) at any level. Use explicit redaction or field exclusion.
- Use `loguru`'s `bind()` and `contextualize()` to attach persistent context (e.g., request-scoped correlation ID) rather than repeating it in every manual call.
- When catching exceptions at boundaries, log the full exception chain with `logger.exception()` or `logger.opt(exception=True).error(...)` before wrapping into `returns.Failure`.
- Ensure log messages are complete sentences with proper capitalization and punctuation. Avoid cryptic abbreviations or single-word messages.
- Example of a professional log entry:
  ```python
  logger.info(
      "User registration completed successfully.",
      user_id=str(user.id),
      email_domain=user.email.split("@")[-1],
      duration_ms=elapsed,
  )
  ```
