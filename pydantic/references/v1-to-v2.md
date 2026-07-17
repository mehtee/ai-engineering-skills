# Pydantic v1 → v2 cheat sheet

## Install

```bash
pip install -U "pydantic>=2"
pip install -U "pydantic-settings>=2"   # if using BaseSettings
```

## API renames

| v1 | v2 |
|----|----|
| `class Config:` | `model_config = ConfigDict(...)` |
| `orm_mode = True` | `from_attributes=True` |
| `allow_mutation = False` | `frozen=True` |
| `@validator` | `@field_validator` (+ `@classmethod`) |
| `@root_validator` | `@model_validator` |
| `@validator(..., pre=True)` | `mode="before"` |
| `.parse_obj(x)` | `.model_validate(x)` |
| `.parse_raw` / `.parse_file` | `.model_validate_json` / read file then validate |
| `.dict()` | `.model_dump()` |
| `.json()` | `.model_dump_json()` |
| `.copy(update=...)` | `.model_copy(update=...)` |
| `.schema()` | `.model_json_schema()` |
| `.construct()` | `.model_construct()` |
| `__root__` / `root` custom root | `TypeAdapter` or normal field |
| `BaseSettings` in `pydantic` | `from pydantic_settings import BaseSettings` |
| `//` constrained types module | `Annotated` + `Field` / `StringConstraints` |
| `getter_dict` etc. | removed — use `from_attributes` |

## Config mapping examples

```python
# v1
class M(BaseModel):
    class Config:
        orm_mode = True
        allow_population_by_field_name = True
        extra = "forbid"

# v2
class M(BaseModel):
    model_config = ConfigDict(
        from_attributes=True,
        populate_by_name=True,
        extra="forbid",
    )
```

## Validator migration

```python
# v1
@validator("name")
def check(cls, v):
    return v.strip()

# v2
@field_validator("name")
@classmethod
def check(cls, v: str) -> str:
    return v.strip()
```

```python
# v1
@root_validator
def check(cls, values):
    ...
    return values

# v2 after
@model_validator(mode="after")
def check(self) -> Self:
    ...
    return self
```

## Generic models

Update `GenericModel` (v1) to `BaseModel, Generic[T]` (v2).

## `.dict()` flag renames

| v1 | v2 |
|----|----|
| `skip_defaults` (deprecated) | `exclude_defaults` |
| `exclude_unset` | same |
| `by_alias` | same |
| `json_encoders` in Config | custom serializers / `PlainSerializer` |

## Breaking behavioral notes

- Stricter type coercion in some cases; enable `strict=True` only intentionally.
- `Optional` defaults: be explicit with `= None`.
- Custom `__init__` discouraged; use validators / `model_validator`.
- `parse_obj_as(T, x)` → `TypeAdapter(T).validate_python(x)`.
- Error `type` strings changed — update tests that assert exact error types.

## Migration strategy

1. Upgrade dependency; run tests.
2. Replace runtime APIs (`parse_obj`, `dict`, `json`) first — high volume.
3. Move `Config` → `ConfigDict`.
4. Rewrite validators.
5. Move settings to `pydantic-settings`.
6. Replace custom root models with `TypeAdapter`.
7. Revisit `json_encoders` → field/model serializers.

Official guide: https://docs.pydantic.dev/latest/migration/
