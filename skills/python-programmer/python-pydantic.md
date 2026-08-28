---
name: python-pydantic

description: >
  Pydantic modeling, validation, and type safety rules for Python.
  All custom types must be Pydantic models or Enums.
  Function I/O contracts must exhaustively use Pydantic models—no bare primitives.
  Use @dataclass(frozen=True) for lightweight internal value objects.
---

# Python Pydantic and Type Safety

## Model Data with Pydantic and @dataclass(frozen=True)

- **All custom types must be defined as Pydantic models (`pydantic.BaseModel`) or Enums.**
- **Function input and output contracts must exhaustively use Pydantic models.** Never use bare primitives (`str`, `int`, `float`, `bool`, `list`, `dict`, etc.) in function signatures. Wrap even single primitive values in a Pydantic model to give them domain meaning.
- Every function return type must be `returns.Result[PydanticModel]` (or `returns.Maybe[PydanticModel]` where absence is expected). No function should return a bare primitive or unwrapped custom type.
- Use `pydantic.Field` for validation constraints (e.g., `min_length`, `ge`, `pattern`) directly on the model definition.
- Use Pydantic validators (`@field_validator`, `@model_validator`) to enforce invariants at object creation.
- **For lightweight internal value objects that do not cross function boundaries**, you may use `@dataclass(frozen=True)` to enforce immutability and value semantics.
- Use `@property` for computed properties on data models.
- Keep data models focused on state and lightweight behavior; move orchestration into functions or services.

## Validation and Type Safety

- Validate external input immediately: request payloads, config, files, environment variables, and user input.
- **Use Pydantic models as the single validation layer** for all incoming data. Define constraints via `Field()` and custom validators so invalid data never reaches domain logic.
- Make invalid states unrepresentable where practical by using narrow types, validated Pydantic constructors, and Enums.
- Add or tighten annotations when touching a module; avoid leaving new `Any`-shaped gaps.
- Prefer `typing` constructs that document intent clearly, such as `TypeAlias`, `Protocol`, `Literal`, and generics when they genuinely help.
- Keep `returns` containers (`Result[PydanticModel]`, `Maybe[PydanticModel]`) and Pydantic fields fully typed so `ty` can catch mismatches early.
- If a cast seems necessary, first look for a better Pydantic type model or validation step.
