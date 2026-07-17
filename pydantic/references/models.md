# Models & fields (Pydantic v2)

## BaseModel essentials

```python
from pydantic import BaseModel, ConfigDict, Field
from typing import Annotated


class Item(BaseModel):
    model_config = ConfigDict(
        extra="forbid",
        str_strip_whitespace=True,
        validate_assignment=True,
        use_enum_values=False,
        ser_json_timedelta="iso8601",
    )

    id: int
    title: Annotated[str, Field(min_length=1, max_length=200)]
    price: Annotated[float, Field(gt=0, description="Unit price")]
    tags: list[str] = Field(default_factory=list)
```

### Construction & validation entry points

| Method | Use |
|--------|-----|
| `Model(**kwargs)` | Python kwargs (validates) |
| `Model.model_validate(obj)` | dict / object / model |
| `Model.model_validate_json(s)` | JSON string/bytes |
| `Model.model_construct(**kwargs)` | **No validation** — trusted internal only |

Prefer `model_validate` over `__init__` when the source is a dict from JSON/ORM.

## Field patterns

```python
from decimal import Decimal
from pydantic import Field, field_serializer
from typing import Annotated

Percent = Annotated[Decimal, Field(ge=0, le=100, decimal_places=2)]

class Product(BaseModel):
    sku: str = Field(pattern=r"^[A-Z]{3}-\d{4}$")
    name: str = Field(examples=["Widget"])
    discount: Percent = Decimal("0")
    metadata: dict[str, str] = Field(default_factory=dict, max_length=50)  # max keys where supported
```

### Defaults

- Immutable default: `x: int = 0`
- Mutable default: **always** `Field(default_factory=list)` / `dict` / `set` — never `= []`
- Required field: no default
- Optional + absent vs null: `x: int | None = None` (optional, may be null) vs required nullability depends on whether default is set

## ConfigDict cheatsheet

| Key | Typical use |
|-----|-------------|
| `extra` | `"forbid"` inputs; `"ignore"` external; `"allow"` passthrough |
| `frozen` | Immutable instances |
| `validate_assignment` | Re-validate on attribute set |
| `str_strip_whitespace` | Trim strings |
| `str_min_length` / `str_max_length` | Global string bounds |
| `populate_by_name` | Accept field name and alias |
| `from_attributes` | ORM / attribute objects (`model_validate(orm_obj)`) |
| `arbitrary_types_allowed` | Non-Pydantic types (prefer custom validators) |
| `use_enum_values` | Store raw enum values |
| `validate_default` | Validate default values too |
| `revalidate_instances` | `"always"` / `"subclass-instances"` |
| `ser_json_bytes` | `"base64"` / `"hex"` |
| `hide_input_in_errors` | Avoid leaking secrets in errors |

```python
class OrmUser(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    email: str
```

## Computed fields

```python
from pydantic import BaseModel, computed_field


class Rectangle(BaseModel):
    width: float
    height: float

    @computed_field
    @property
    def area(self) -> float:
        return self.width * self.height
```

Included in `model_dump` by default. Use `cached_property` only when expensive and instance is stable.

## Inheritance

- Share mixins as abstract-ish base models (common fields + config).
- Override fields carefully; narrower types OK when intentional.
- Prefer composition (nested models) over deep inheritance trees.

```python
class Timestamped(BaseModel):
    created_at: datetime
    updated_at: datetime | None = None


class Post(Timestamped):
    title: str
    body: str
```

## Nested models & collections

```python
class Address(BaseModel):
    city: str
    country: Annotated[str, Field(min_length=2, max_length=2)]


class Customer(BaseModel):
    name: str
    addresses: list[Address]
    primary: Address | None = None
```

## Private attributes

Not fields — not validated as input, not in schema by default:

```python
from pydantic import BaseModel, PrivateAttr


class Worker(BaseModel):
    name: str
    _cache: dict[str, object] = PrivateAttr(default_factory=dict)
```

## Model copy / update

```python
updated = item.model_copy(update={"price": 9.99}, deep=True)
```

With `frozen=True`, always use `model_copy` instead of mutation.

## JSON schema

```python
schema = Item.model_json_schema()
```

Customize with `Field(json_schema_extra=...)` or `model_config = ConfigDict(json_schema_extra=...)`.

## Enums

```python
from enum import Enum


class Role(str, Enum):
    admin = "admin"
    user = "user"


class Account(BaseModel):
    role: Role = Role.user
```

Prefer `str, Enum` for JSON-friendly APIs.

## Dataclass interop

Pydantic can validate dataclasses via `TypeAdapter` or `pydantic.dataclasses.dataclass` when you need validation on stdlib-style classes. Prefer `BaseModel` for API boundaries.
