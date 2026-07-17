# Advanced types (Pydantic v2)

## TypeAdapter

Validate/serialize values that are not a single BaseModel:

```python
from pydantic import TypeAdapter, BaseModel


class User(BaseModel):
    id: int
    name: str


Users = TypeAdapter(list[User])
users = Users.validate_python([{"id": 1, "name": "Ada"}])
json_bytes = Users.dump_json(users)
schema = Users.json_schema()
```

Also: `TypeAdapter(int | str)`, `TypeAdapter(dict[str, User])`.

Cache adapters at module level — building them has cost.

## Generics

```python
from typing import Generic, TypeVar
from pydantic import BaseModel

T = TypeVar("T")


class Page(BaseModel, Generic[T]):
    items: list[T]
    total: int


class Item(BaseModel):
    id: int


page: Page[Item] = Page[Item].model_validate(
    {"items": [{"id": 1}], "total": 1}
)
```

## Discriminated unions

```python
from typing import Annotated, Literal, Union
from pydantic import BaseModel, Field


class Cat(BaseModel):
    kind: Literal["cat"]
    meow_volume: int


class Dog(BaseModel):
    kind: Literal["dog"]
    bark_volume: int


Pet = Annotated[Union[Cat, Dog], Field(discriminator="kind")]


class Shelter(BaseModel):
    pets: list[Pet]
```

Better errors and faster matching than bare unions. Nested discriminators supported via `Discriminator` callable for complex cases.

## Tagged / ADT style APIs

```python
Event = Annotated[
    Union[UserCreated, UserDeleted, UserUpdated],
    Field(discriminator="type"),
]
adapter = TypeAdapter(Event)
```

## Root-like models

v1 `__root__` is removed. Use:

```python
adapter = TypeAdapter(list[User])
# or a named model:
class UserList(BaseModel):
    root: list[User]  # ordinary field — or just use TypeAdapter
```

For JSON that is a bare array, `TypeAdapter(list[User])` is the idiomatic choice.

## Callables / arbitrary types

```python
from pydantic import BaseModel, ConfigDict, GetCoreSchemaHandler
from pydantic_core import core_schema


class IpAddress:
    def __init__(self, v: str):
        self.v = v

    @classmethod
    def __get_pydantic_core_schema__(cls, source, handler: GetCoreSchemaHandler):
        return core_schema.no_info_after_validator_function(
            cls,
            core_schema.str_schema(),
        )
```

Prefer this over broad `arbitrary_types_allowed=True`.

## Network & special types

From `pydantic`:

- `EmailStr` (needs `email-validator`)
- `NameEmail`
- `AnyUrl`, `HttpUrl`, `AnyHttpUrl`, `FileUrl`, `PostgresDsn`, `RedisDsn`, …
- `UUID1` / `UUID3` / `UUID4` / `UUID5` (or stdlib `UUID`)
- `PastDate`, `FutureDate`, `NaiveDatetime`, `AwareDatetime`
- `ByteSize`
- `Json` / `JsonValue`
- `SecretStr`, `SecretBytes`
- `StrictBool`, `StrictInt`, … or `Field(strict=True)` / `ConfigDict(strict=True)`

## Defer building (large graphs)

```python
class Node(BaseModel):
    model_config = ConfigDict(defer_build=True)
    children: list["Node"] = []
```

Useful for heavy import-time graphs.

## JSON Schema customisation

```python
class Model(BaseModel):
    x: int

    model_config = ConfigDict(
        json_schema_extra={"examples": [{"x": 1}]},
    )
```

Or per-field `Field(json_schema_extra=...)`.

## Validate call (functions)

```python
from pydantic import validate_call


@validate_call
def add(a: int, b: int) -> int:
    return a + b
```

Good for CLI/tool boundaries; avoid on hot inner loops.
