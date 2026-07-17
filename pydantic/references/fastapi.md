# FastAPI + Pydantic v2

FastAPI uses Pydantic models for request bodies, query/path params (simple types),
response models, and OpenAPI generation.

## Request / response split

```python
from pydantic import BaseModel, ConfigDict, EmailStr, Field


class UserCreate(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)
    email: EmailStr
    name: str = Field(min_length=1, max_length=100)


class UserRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    email: EmailStr
    name: str


@app.post("/users", response_model=UserRead, status_code=201)
def create_user(body: UserCreate) -> UserRead:
    row = repo.create(**body.model_dump())
    return UserRead.model_validate(row)
```

Rules:

- **Input models**: `extra="forbid"`, tight constraints, no internal IDs unless needed.
- **Output models**: only fields safe to expose; `from_attributes=True` for ORM.
- Never reuse one model for create + DB + response if fields differ (password hash, flags).

## Response model vs return type

Prefer both aligned:

```python
@app.get("/users/{user_id}", response_model=UserRead)
def get_user(user_id: int) -> UserRead:
    ...
```

`response_model` filters output; return type helps type checkers.

## Nested bodies & embed

```python
class Item(BaseModel):
    name: str

@app.post("/items")
def create(item: Item):  # body = {"name": "..."}
    ...
```

## Partial updates (PATCH)

```python
class UserUpdate(BaseModel):
    model_config = ConfigDict(extra="forbid")
    name: str | None = None
    email: EmailStr | None = None


@app.patch("/users/{user_id}", response_model=UserRead)
def patch_user(user_id: int, body: UserUpdate) -> UserRead:
    data = body.model_dump(exclude_unset=True)
    row = repo.update(user_id, **data)
    return UserRead.model_validate(row)
```

`exclude_unset=True` is essential so omitted fields are not nulled.

## ORM mode

```python
class UserRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    email: str
```

SQLAlchemy 2: return ORM instance; FastAPI/Pydantic reads attributes.

## Dependencies + settings

See [settings.md](settings.md) for `get_settings` + `lru_cache`.

## Error format

FastAPI already converts `ValidationError` on request bodies to 422 with `detail` list.
For manual validation:

```python
from fastapi import HTTPException
from pydantic import ValidationError

try:
    model = Model.model_validate(raw)
except ValidationError as e:
    raise HTTPException(status_code=422, detail=e.errors(include_url=False))
```

## OpenAPI examples

```python
class Payload(BaseModel):
    model_config = ConfigDict(
        json_schema_extra={
            "examples": [{"name": "Ada", "email": "ada@example.com"}]
        }
    )
    name: str
    email: EmailStr
```

Or `Field(examples=[...])` / FastAPI `openapi_examples` on parameters.

## Background: version alignment

- FastAPI recent versions require Pydantic v2.
- If a legacy app is on Pydantic v1, do not introduce v2-only APIs; plan migration
  ([v1-to-v2.md](v1-to-v2.md)).
