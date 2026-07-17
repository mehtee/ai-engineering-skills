# Best practices & anti-patterns

## Do

1. **Bound inputs.** Public models use constraints (`Field`, `Annotated`, `StringConstraints`) and usually `extra="forbid"`.
2. **Separate models by role.** `Create` / `Update` / `Read` / `DB` / `Internal` — do not overload one class with optional password + hash + admin flags.
3. **Pure validators.** No DB/HTTP/filesystem in validators. Validate shape → service layer for business rules.
4. **`default_factory` for mutables.** Never `tags: list[str] = []`.
5. **Secrets as `SecretStr`.** Prevents accidental log leakage via `repr` / dump.
6. **Explicit version.** Target Pydantic v2 APIs; check `pydantic.VERSION` in mixed repos.
7. **Stable TypeAdapters.** Module-level `TypeAdapter(...)` for hot paths.
8. **Discriminators on unions.** Faster, clearer 422s.
9. **`model_dump(exclude_unset=True)` for PATCH.** Preserve omitted fields.
10. **Fail fast settings.** Required env without defaults at process start.
11. **Match project conventions.** Alias generators, datetime TZ policy, error shape.
12. **Document non-obvious invariants** in `Field(description=...)` — feeds OpenAPI.

## Don't

| Anti-pattern | Prefer |
|--------------|--------|
| v1 `class Config` in new code | `model_config = ConfigDict(...)` |
| `@validator` / `@root_validator` | `@field_validator` / `@model_validator` |
| `.dict()` / `.json()` / `.parse_obj` | `.model_dump()` / `.model_dump_json()` / `.model_validate` |
| Catching `Exception` around parse | `except ValidationError` |
| Business logic inside validators | Service / domain functions |
| One mega-model for all layers | Layered models |
| `arbitrary_types_allowed` as default | Custom core schema or convert at boundary |
| Logging full validation `input` for auth fields | Redact / `hide_input_in_errors` |
| Nested `model_validate` in tight loops without need | Batch with `TypeAdapter` |
| Mutable global Settings mutated at runtime | Immutable settings; restart or explicit reload |

## Model layering (recommended)

```text
HTTP JSON  →  UserCreate (forbid, tight)
           →  domain function / service
           →  ORM / repository
           →  UserRead (from_attributes, public fields)
           →  HTTP JSON
```

## Performance notes

- Prefer `model_validate` on already-parsed dicts over repeated JSON parse.
- Reuse models and adapters; avoid dynamic `create_model` in request path unless necessary.
- `frozen=True` helps share instances safely.
- Strict mode avoids coercion cost/surprise when clients are controlled.
- Large lists: `TypeAdapter(list[Model]).validate_python(data)` once.

## Testing patterns

```python
import pytest
from pydantic import ValidationError


def test_rejects_bad_email():
    with pytest.raises(ValidationError) as ei:
        UserCreate.model_validate({"email": "not-an-email", "name": "x"})
    assert any(e["loc"] == ("email",) for e in ei.value.errors())


def test_accepts_minimal():
    u = UserCreate.model_validate({"email": "a@b.co", "name": "Ada"})
    assert u.name == "Ada"
```

Parametrize valid/invalid tables for constraint-heavy models.

## Security checklist

- [ ] `extra="forbid"` on untrusted input
- [ ] Password/token fields are `SecretStr` and excluded from responses
- [ ] No sensitive data in `Field(examples=...)`
- [ ] Settings secrets not committed; `.env` gitignored
- [ ] Size limits on strings/lists to reduce abuse
- [ ] URL fields use constrained types (`HttpUrl`) when only HTTP(S) is intended

## Style defaults (when greenfield)

```python
model_config = ConfigDict(
    extra="forbid",
    str_strip_whitespace=True,
    validate_assignment=True,
    from_attributes=False,  # True only on read/ORM models
)
```

CamelCase JSON APIs: `alias_generator=to_camel` + `populate_by_name=True`.
