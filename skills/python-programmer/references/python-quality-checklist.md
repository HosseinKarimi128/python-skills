# Python Quality Checklist

Use this file as a quick implementation checklist when applying the skill.

## Modeling

- Use `attrs` for domain objects and input containers.
- Freeze value objects when mutation is unnecessary.
- Keep invariants close to construction with validators and converters.
- Prefer explicit domain types over loose dictionaries.

## Control Flow

- Prefer `returns.result.Result` for expected failures.
- Define structured error types before reaching for broad exception handling.
- Convert external exceptions into domain failures at boundaries.
- Keep decision-making in pure functions when practical.

## Typing

- Add precise annotations for every new public function.
- Avoid introducing new `Any`, implicit `Optional`, or ambiguous unions.
- Prefer protocols or aliases when they reduce repetition and clarify intent.
- Remove casts by improving validation or type modeling first.

## Code Shape

- Prefer clear abstractions over repetitive boilerplate.
- Keep modules cohesive and dependency direction obvious.
- Separate I/O, parsing, domain logic, and formatting.
- Avoid flag arguments and large god-functions.

## Verification

- Run targeted `pytest` coverage for changed behavior.
- Check `Success` and `Failure` paths explicitly.
- Run `ruff` and `ty` on touched files when available.
- Document any validation you could not run.
