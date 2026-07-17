# Serialization & aliases (Pydantic v2)

## Dump APIs

```python
user.model_dump()                      # Python objects
user.model_dump(mode="json")           # JSON-ready (datetime→str, etc.)
user.model_dump_json()                 # str
user.model_dump(include={"id", "email"})
user.model_dump(exclude={"password"})
user.model_dump(exclude_none=True)
user.model_dump(exclude_unset=True)    # only fields explicitly set
user.model_dump(exclude_defaults=True)
user.model_dump(by_alias=True)
```

| Flag | Meaning |
|------|---------|
| `exclude_unset` | PATCH-friendly: omit fields never assigned |
| `exclude_none` | Drop nulls |
| `exclude_defaults` | Drop fields still at default |
| `by_alias` | Use serialization aliases |
| `mode="json"` | Coerce to JSON types |

## Aliases

```python
from pydantic import BaseModel, ConfigDict, Field, AliasChoices, AliasPath


class ApiUser(BaseModel):
    model_config = ConfigDict(populate_by_name=True)

    user_id: int = Field(alias="userId")
    # accept several input keys:
    name: str = Field(validation_alias=AliasChoices("name", "full_name", "fullName"))
    # nested extract:
    city: str | None = Field(default=None, validation_alias=AliasPath("address", "city"))
```

Alias generators:

```python
from pydantic import ConfigDict
from pydantic.alias_generators import to_camel


class CamelModel(BaseModel):
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True,
        ser_json_by_alias=True,  # where available / use by_alias=True on dump
    )
    first_name: str
    last_name: str
```

Split validation vs serialization aliases when wire format differs from internal names:

```python
email: str = Field(validation_alias="emailAddress", serialization_alias="email")
```

## Custom serializers

```python
from datetime import datetime, timezone
from pydantic import field_serializer, model_serializer


class Event(BaseModel):
    ts: datetime
    payload: dict[str, object]

    @field_serializer("ts")
    def ser_ts(self, v: datetime) -> str:
        return v.astimezone(timezone.utc).isoformat()

    @model_serializer(mode="wrap")
    def ser_model(self, handler):
        data = handler(self)
        data["v"] = 1
        return data
```

Modes: `plain` / `wrap`; also `when_used` (`always`, `unless-none`, `json`, `json-unless-none`).

## Computed field serialization

`@computed_field` values appear in dumps. Exclude with:

```python
@computed_field(repr=False)
@property
def secret_derived(self) -> str: ...
```

Or `model_dump(exclude={"secret_derived"})`.

## Secrets

```python
from pydantic import BaseModel, SecretStr


class Login(BaseModel):
    username: str
    password: SecretStr

creds = Login(username="a", password="hunter2")
creds.password.get_secret_value()   # real value
creds.model_dump()                  # password -> SecretStr('**********')
```

Use `SecretStr` / `SecretBytes` for tokens and passwords so logs and `repr` do not leak.

## Encode custom types

```python
from pydantic import PlainSerializer
from typing import Annotated
from datetime import date

IsoDate = Annotated[
    date,
    PlainSerializer(lambda d: d.isoformat(), return_type=str, when_used="json"),
]
```

## Round-trip recipe

```python
data = model.model_dump(mode="json", by_alias=True)
restored = Model.model_validate(data)
# or
text = model.model_dump_json(by_alias=True)
restored = Model.model_validate_json(text)
```

Ensure aliases and custom serializers are invertible if you need round-trips.

## Pickle / deepcopy

Prefer `model_dump` + `model_validate` for process boundaries. `model_copy(deep=True)` for in-process clones.
