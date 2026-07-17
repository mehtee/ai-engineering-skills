# Validation (Pydantic v2)

## Constraint Field / Annotated

```python
from typing import Annotated
from pydantic import BaseModel, Field, StringConstraints

Username = Annotated[
    str,
    StringConstraints(strip_whitespace=True, min_length=3, max_length=32, pattern=r"^[a-z0-9_]+$"),
]

class Signup(BaseModel):
    username: Username
    age: Annotated[int, Field(ge=13, le=120)]
    score: Annotated[float, Field(gt=0)]
```

Common numeric: `gt`, `ge`, `lt`, `le`, `multiple_of`.  
Common string: `min_length`, `max_length`, `pattern`.  
Collections: `min_length`, `max_length` on `list`/`dict` fields.

## field_validator

```python
from pydantic import BaseModel, field_validator


class User(BaseModel):
    email: str
    name: str

    @field_validator("email")
    @classmethod
    def normalize_email(cls, v: str) -> str:
        return v.lower().strip()

    @field_validator("name", mode="before")
    @classmethod
    def empty_to_default(cls, v: object) -> object:
        if v is None or v == "":
            return "Anonymous"
        return v
```

Modes:

| Mode | When | Input |
|------|------|--------|
| `after` (default) | After type coercion | typed value |
| `before` | Raw input | any |
| `plain` | No further validation | any → final |
| `wrap` | Around inner validation | handler callback |

Always `@classmethod`. Raise `ValueError` / `AssertionError` / `PydanticCustomError` for failures.

### Multiple fields

```python
@field_validator("start", "end")
@classmethod
def not_sentinel(cls, v: int) -> int:
    if v == -1:
        raise ValueError("sentinel not allowed")
    return v
```

## model_validator

```python
from pydantic import BaseModel, model_validator
from typing import Self


class DateRange(BaseModel):
    start: int
    end: int

    @model_validator(mode="after")
    def check_order(self) -> Self:
        if self.end < self.start:
            raise ValueError("end must be >= start")
        return self

    @model_validator(mode="before")
    @classmethod
    def accept_pair(cls, data: object) -> object:
        if isinstance(data, (list, tuple)) and len(data) == 2:
            return {"start": data[0], "end": data[1]}
        return data
```

- `before`: reshape raw input (dict/list/etc.)
- `after`: cross-field checks on constructed model (`Self`)
- `wrap`: advanced wrap around whole validation

## Annotated validators (reusable)

```python
from typing import Annotated
from pydantic import AfterValidator, BeforeValidator, WrapValidator


def _strip(v: str) -> str:
    return v.strip()

def _nonempty(v: str) -> str:
    if not v:
        raise ValueError("empty")
    return v

CleanName = Annotated[str, BeforeValidator(_strip), AfterValidator(_nonempty)]
```

Prefer these for **reusable** types shared across models; keep model-specific rules as methods.

## ValidationInfo

```python
from pydantic import ValidationInfo, field_validator


class Row(BaseModel):
    unit: str
    value: float

    @field_validator("value")
    @classmethod
    def unit_aware(cls, v: float, info: ValidationInfo) -> float:
        # info.data has already-validated fields (order matters)
        unit = info.data.get("unit")
        if unit == "percent" and not 0 <= v <= 100:
            raise ValueError("percent out of range")
        return v
```

Field order in the model affects `info.data` completeness for earlier fields.

## Custom errors

```python
from pydantic_core import PydanticCustomError


raise PydanticCustomError(
    "username_reserved",
    "Username {name} is reserved",
    {"name": v},
)
```

## Handling ValidationError

```python
from pydantic import ValidationError

try:
    User.model_validate(payload)
except ValidationError as e:
    print(e.error_count())
    for err in e.errors():
        # loc, msg, type, input, ctx
        print(err["loc"], err["type"], err["msg"])
    # API response:
    return {"detail": e.errors(include_url=False, include_input=False)}
```

Never log full `input` for password/secret fields — use `hide_input_in_errors` or strip in the API layer.

## Strict vs lax

```python
from pydantic import BaseModel, ConfigDict, Field
from typing import Annotated

class StrictIds(BaseModel):
    model_config = ConfigDict(strict=True)
    id: int  # "1" will fail

class Mixed(BaseModel):
    id: Annotated[int, Field(strict=True)]
    count: int  # lax coercion still allowed
```

Use strict at trust boundaries when clients must send correct JSON types.

## Instance checks

```python
from pydantic import TypeAdapter

adapter = TypeAdapter(list[User])
users = adapter.validate_python(raw_list)
```

## Async validation

Pydantic core validation is sync. For async checks (unique email in DB):

1. Validate shape with Pydantic first.
2. Run async business rules in the service layer.
3. Or use FastAPI dependencies — do not block event loop inside validators with sync I/O.

## Functional validators vs methods

| Prefer | When |
|--------|------|
| `Annotated` + `AfterValidator` | Shared branded types |
| `@field_validator` | One model, multi-field names, needs `ValidationInfo` |
| `@model_validator` | Cross-field / reshape whole payload |
