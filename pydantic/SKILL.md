---
name: pydantic
description: >
  Pydantic v2 models, validation, serialization, settings, and TypeAdapter for Python.
  Use when defining BaseModel/BaseSettings, writing field/model validators, configuring
  model_config, serializing/deserializing JSON, migrating from Pydantic v1, integrating
  with FastAPI/Django/SQLAlchemy, or debugging ValidationError. Prefer this skill over
  inventing schemas from memory.
---

# Pydantic (v2)

Authoritative patterns for **Pydantic v2** in Python. Assume v2 unless the project pins
`pydantic<2`. Do not mix v1 APIs (`class Config`, `@validator`, `.dict()`, `.parse_obj`)
into new code.

## When this skill applies

- Designing or refactoring request/response DTOs, domain models, configs
- Custom validation (field-level, cross-field, async, after-mode)
- JSON schema / OpenAPI shape control
- `pydantic-settings` for env/dotenv/secrets
- Performance (strict types, `TypeAdapter`, reuse of validators)
- v1 → v2 migration

## Progressive disclosure — read only what you need

| Topic | File |
|-------|------|
| Models, fields, `model_config`, computed fields | [references/models.md](references/models.md) |
| Validators, constraints, custom errors | [references/validation.md](references/validation.md) |
| Dump/serialize, aliases, JSON schema | [references/serialization.md](references/serialization.md) |
| Settings / env config (`pydantic-settings`) | [references/settings.md](references/settings.md) |
| TypeAdapter, generics, discriminated unions | [references/advanced-types.md](references/advanced-types.md) |
| FastAPI / web integration | [references/fastapi.md](references/fastapi.md) |
| Best practices & anti-patterns | [references/best-practices.md](references/best-practices.md) |
| v1 → v2 cheat sheet | [references/v1-to-v2.md](references/v1-to-v2.md) |
| External docs index | [references/links.md](references/links.md) |

**Default workflow:** open `best-practices.md` + the one topic file for the task. Do not
load every reference file unless the task spans multiple areas.

## Quick start (canonical v2)

```python
from datetime import datetime
from typing import Annotated
from pydantic import BaseModel, ConfigDict, EmailStr, Field, field_validator


class UserCreate(BaseModel):
    model_config = ConfigDict(
        str_strip_whitespace=True,
        validate_assignment=True,
        extra="forbid",
    )

    email: EmailStr
    name: Annotated[str, Field(min_length=1, max_length=100)]
    age: int | None = Field(default=None, ge=0, le=150)

    @field_validator("name")
    @classmethod
    def name_not_blank(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("name must not be blank")
        return v


class User(UserCreate):
    id: int
    created_at: datetime
```

Parse / dump:

```python
user = User.model_validate({"id": 1, "email": "a@b.co", "name": "Ada", "created_at": "2024-01-01T00:00:00Z"})
payload = user.model_dump(mode="json")          # JSON-friendly primitives
json_text = user.model_dump_json()
clone = User.model_validate_json(json_text)
```

## Decision rules (apply first)

1. **v2 APIs only** in new code: `model_validate`, `model_dump`, `model_config = ConfigDict(...)`,
   `@field_validator` / `@model_validator`, `Field`, `Annotated[...]`.
2. **`extra="forbid"`** on public input models (API bodies, CLI args, settings that should not
   silently accept typos). Use `ignore` only for forward-compat external payloads you do not own.
3. Prefer **`Annotated[T, Field(...)]`** (or plain `Field` defaults) over unconstrained types
   for user-facing fields.
4. Cross-field rules → `@model_validator(mode="after")`. Single-field pure transforms →
   `@field_validator` or `AfterValidator` / `BeforeValidator` in `Annotated`.
5. Config from environment → **`pydantic-settings`** `BaseSettings`, not hand-rolled `os.getenv`.
6. Non-model parse targets (list of models, union root) → **`TypeAdapter`**, not fake root models
   (v1 `root` / `__root__` is gone).
7. Never catch bare `Exception` around validation; catch **`ValidationError`** and surface
   `e.errors()`.
8. Keep domain models **immutable when shared** (`frozen=True`) if instances are cached or used
   as dict keys; keep mutable DTOs for request builders when needed.
9. Do not put heavy I/O (DB, HTTP) inside validators; validators must stay pure/fast and
   side-effect free unless the product explicitly requires otherwise.
10. Match existing project style when editing a codebase that already chose conventions
    (e.g. `populate_by_name`, alias generators); do not fight local standards.

## Install matrix

```bash
pip install "pydantic>=2.0"
pip install "pydantic[email]"          # EmailStr
pip install "pydantic-settings>=2.0"   # BaseSettings
pip install "pydantic[timezone]"       # zoneinfo helpers where needed
```

Check version in-repo before advising migration:

```bash
python -c "import pydantic; print(pydantic.VERSION)"
```

## Output expectations when using this skill

When writing or reviewing Pydantic code:

- State Pydantic major version assumed.
- Prefer complete, copy-pasteable models over fragments.
- Show `ValidationError` shapes when explaining failures.
- Call out security-sensitive settings (`SecretStr`, env file permissions, `extra` policy).
- Link to the specific `references/*.md` section you followed if the user may need depth.

## Anti-triggers (do not force this skill)

- Pure typing / dataclasses with no validation need and no Pydantic dependency
- ORM schema design with no Pydantic layer (unless user asks for Pydantic models on top)
- Non-Python stacks
