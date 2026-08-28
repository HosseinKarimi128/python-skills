---
name: python-functional

description: >
  Functional programming principles for Python: pure functions, function composition,
  returns usage (Result, Maybe, combinators), Protocol-based traits, pattern matching
  with match-case, and explicit error types. Prefer functions over classes.
---

# Python Functional Programming

## Functional Programming First

- **Prefer functions over classes**. Encode behavior as small, pure, composable functions rather than class methods.
- Keep functions short enough to read in one pass. Extract pure functions from mixed side-effect code before larger reorganizations.
- Use function composition and piping (via `returns`) to chain transformations left-to-right.
- **Write fluent chains vertically.** Never collapse chained calls onto one line like `obj.st().st().st()`; format as:
  ```python
  obj
  .st()
  .st()
  .st()
  ```
- **Write multiline call arguments one-per-line** when there are multiple arguments or the call is not trivially short; use:
  ```python
  func(
      arg1,
      arg2,
      arg3,
  )
  ```
- Keep side effects at the edges: file access, network calls, database operations, and CLI/UI behavior.
- Avoid hidden mutation; return new values when that improves predictability.
- Avoid boolean flag arguments when separate functions or richer types describe intent better.
- Replace repeated conditional or transformation logic with small helpers or domain-specific functions.
- Encode invariants in constructors, validation functions, and dedicated value objects instead of scattering checks.
- Prefer explicit domain terms over generic utility names.

## Traits via Protocol with @runtime_checkable

- Define abstract behavior using `typing.Protocol` decorated with `@runtime_checkable`.
- Protocol methods should contain only `...` (or `pass`) with fully typed signatures—this models **traits** as in Rust.
- Use protocols to generalize behavior across types without forcing inheritance.
- When a concrete class must implement a trait (protocol), define it with `@attrs.define` to avoid writing `__init__` boilerplate.
- Do not use traditional inheritance hierarchies; prefer structural subtyping via protocols.

## Pattern Matching with match-case

- **Prefer `match-case` over deep `if-elif-else` chains**.
- Define all possible patterns for domain branching as `Enum` classes (or tagged unions) so the compiler and reader can see the full decision space.
- Use pattern matching to destructure dataclasses, enums, and tuples at the point of decision.
- Ensure `match` statements are exhaustive; if a fallback case is truly unreachable, explicitly mark it with a comment and a narrow `raise`.

## Use returns for All Functional Operations

- **YOU HAVE TO USE the `returns` library** for all functional composition, piped operations, error handling, and optional value management.
- Leverage `returns` capabilities including `Maybe`, `Result`, `Success`, `Failure`, `map`, `bind`, `alt`, `unwrap`, `lash`, `value_or`, `or_else_call`, and other composable helpers.
- Handle optional/absent values with `returns.Maybe` instead of raw `None` checks.
- **Never place `try/except` blocks inside fluent chain/pipeline expressions.**
- **If an operation can raise, isolate it in a small boundary helper that returns `returns.Result` or `returns.Maybe`, then consume it through combinators (`bind`, `map`, `alt`, `lash`, `bind_optional`).**
- Convert exceptions from I/O boundaries into `returns.Result`/`returns.Maybe` containers immediately and continue with chained combinators only.
- **Function return types must always be `returns.Result[PydanticModel]`** — never bare primitives or unwrapped types.
- Code as you would in Rust: explicit types, no hidden mutation, trait-based polymorphism, and exhaustive pattern matching.
- For all fluent/pipeline-style expressions, keep each chained `.` operation on its own line for readability and diff safety.

## Use returns for Explicit Success and Failure Flows

- Prefer `returns.Result[PydanticModel]` (with `Success` and `Failure` variants) for operations that can fail in expected ways.
- Chain transformations with `returns` combinators (`map`, `bind`, `alt`, `lash`, `unwrap`) instead of deeply nested `if` blocks.
- Reserve exceptions for truly exceptional conditions, external-library boundaries, or unrecoverable startup issues.
- **`try/except` is allowed only at boundary-wrapper function level; never as the primary control-flow mechanism in domain pipelines.**
- Return well-defined error types inside `returns.Failure` instead of plain strings whenever the caller may branch on the failure.
- Chain `returns.Result` and `returns.Maybe` inside pipe chains using `bind`, `map`, and `lash`.
- **Never return bare primitives from functions.** Wrap every success value in a Pydantic model before returning it inside `returns.Success`.

## Define Explicit Error Types

- Create small error classes or tagged error values for meaningful failure cases.
- Give each error enough structure to support logging, testing, and caller decisions.
- Keep error variants domain-oriented, such as validation, parsing, lookup, or dependency failures.
- Do not use broad `except Exception` unless re-raising or wrapping at a clear boundary.
- Catch the narrowest expected exception type in wrapper helpers, map to typed domain errors, and return `Failure`/`Nothing` for downstream chain handling.
