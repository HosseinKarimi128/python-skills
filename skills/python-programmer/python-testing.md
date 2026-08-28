---
name: python-testing

description: >
  pytest workflow and test patterns for Python projects using returns.
  Test both Success and Failure paths, Some/Nothing branches, and assert on structured
  error values rather than message text.
---

# Python Testing Workflow

## Testing Workflow

- Add or update `pytest` tests when behavior changes, regressions are plausible, or validation/error handling is introduced.
- Prefer focused unit tests around pure logic and small integration tests at boundaries.
- Test both `Success` and `Failure` paths for `returns.Result`-based flows.
- Test `Some`/`Nothing` branches for `returns` pipelines.
- Assert that functions return `returns.Result[PydanticModel]` and that the wrapped Pydantic model contains the expected fields.
- Assert on structured error values, not only on message text.
- Keep fixtures small and local unless reuse clearly improves readability.
- Run the narrowest relevant test selection first, then expand if the risk profile warrants it.
