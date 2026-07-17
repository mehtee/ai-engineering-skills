# Redis OM (redis-om for Python)

**Mental model** (if you know Django ORM / SQLAlchemy):

> **Pydantic models + Redis persistence + RediSearch indexes + ORM-like query syntax**

It is **not** a relational ORM. No joins, no Alembic/Django-style migrations, no SQL transactions, no lazy relationships. It is a high-level object mapper over **Redis Hashes** and **RedisJSON** with validation and expressive queries.

Stack underneath:

```
Application
    → Redis OM (Pydantic validation + model API)
        → redis-py commands
            → Hashes | RedisJSON | RediSearch
                → Redis / Redis Stack
```

Official docs: [redis.github.io/redis-om-python](https://redis.github.io/redis-om-python/) · Tutorial: [redis.io OM Python](https://redis.io/tutorials/develop/python/redis-om/) · Intro: [Redis blog](https://redis.io/blog/introducing-redis-om-for-python/)

## When redis-om fits

- User profiles, sessions-plus, carts, feature flags, config, chat metadata
- Structured cache objects with validation
- Document-style APIs and real-time dashboards
- AI agent memory / metadata with secondary filters
- Key/document access patterns where rebuild or TTL is acceptable

## When not to use it

- Banking ledgers, ERP, heavy relational schemas
- Complex reporting / joins / multi-row ACID
- Simple string cache → raw `SET`/`GET` is enough
- Max throughput hot path where OM validation overhead matters → raw redis-py
- Vanilla Redis **without** Search: CRUD works; `find()` / indexes / sort do **not**

## Install & runtime

```bash
pip install redis-om
# or: uv add redis-om
```

- Python ≥ 3.9
- Redis always for CRUD
- **Redis Stack** (or RedisJSON + RediSearch) for indexes, `find()`, sort, full-text
- Quick start: `docker run -p 6379:6379 -p 8001:8001 redis/redis-stack` (Insight on :8001)

```bash
export REDIS_OM_URL=redis://localhost:6379/0
```

**Important:** Indexing / migrator work on Redis **logical database 0**. Other DB numbers can raise `MigrationError` on `Migrator().run()`.

Without Search you lose: indexes, `find()`, filtering, sorting. CRUD still works.

## HashModel vs JsonModel

| | HashModel | JsonModel |
|---|---|---|
| Storage | Redis HASH (`HSET`) | RedisJSON document |
| Nested objects | No natural nesting | Nested / lists / dicts |
| Embedded models | — | `EmbeddedJsonModel` |
| Best for | Flat, small, fastest field access | Modern nested documents |
| Default pick | Legacy / flat only | **Prefer for new apps** |

```python
from redis_om import HashModel, JsonModel, EmbeddedJsonModel, Field
```

## Declaring models

Every Redis OM model is also a **Pydantic** model: type coercion + validation on construct/save paths.

```python
import datetime
from typing import Optional
from pydantic import EmailStr
from redis_om import HashModel

class Customer(HashModel):
    first_name: str
    last_name: str
    email: EmailStr
    join_date: datetime.date
    age: int
    bio: Optional[str] = None
```

Supported annotations include: `EmailStr`, `UUID`, `datetime` / `date`, `Decimal`, `Enum`, `Literal`, `list[...]`, `Optional`, custom Pydantic validators.

### Class-level `index=True`

Newer style indexes **all** fields (or use per-field `Field(index=True)`):

```python
class Customer(HashModel, index=True):
    first_name: str
    last_name: str
    email: EmailStr
    age: int
```

Prefer **per-field** indexes in production to control memory:

```python
class Customer(JsonModel):
    email: str = Field(index=True)
    age: int = Field(index=True)
    bio: str = Field(index=True, full_text_search=True, default="")
```

### Meta / connection

```python
from redis import Redis
from redis_om import get_redis_connection

redis = Redis(host="localhost", port=6379, decode_responses=True)
# or: redis = get_redis_connection()  # uses REDIS_OM_URL

class Customer(HashModel, index=True):
    name: str

    class Meta:
        database = redis
        global_key_prefix = "myapp"
        model_key_prefix = "customer"
```

Each model can use a different `Meta.database`.

### Azure Managed Redis (Entra ID)

```bash
pip install redis-entraid
```

```python
from redis import Redis
from redis_entraid.cred_provider import create_from_default_azure_credential
from redis_om import HashModel

credential_provider = create_from_default_azure_credential(
    ("https://redis.azure.com/.default",),
)
db = Redis(
    host="cluster-name.region.redis.azure.net",
    port=10000,
    ssl=True,
    ssl_cert_reqs=None,
    credential_provider=credential_provider,
)

class User(HashModel, index=True):
    first_name: str
    last_name: str

    class Meta:
        database = db
```

## Primary keys

- Field: **`pk`** (auto)
- Format: **ULID** (sortable, globally unique, distributed-friendly)
- Generated client-side before Redis talk

```python
user = Customer(...)
print(user.pk)  # before or after save
user.save()
```

## CRUD

```python
# Create
andrew = Customer(
    first_name="Andrew",
    last_name="Brookins",
    email="andrew.brookins@example.com",
    join_date=datetime.date.today(),
    age=38,
    bio="Python developer",
)
andrew.save()

# Read
assert Customer.get(andrew.pk) == andrew

# Update
andrew.age = 39
andrew.save()

# Delete
andrew.delete()  # instance
# or model-level delete if available in your version: Customer.delete(pk)
```

### TTL / expire

```python
andrew.save()
andrew.expire(120)  # seconds — entire key
```

## Migrator (indexes)

After defining indexed models (and after index-field changes):

```python
from redis_om import Migrator

Migrator().run()  # issues FT.CREATE / updates as needed
# CLI: migrate tool also exists in the project
```

Run once at deploy / app startup (FastAPI lifespan or Django `AppConfig.ready`, careful with reloader double-run).

## Query API (RediSearch)

Requires indexes + modules. Expressions inspired by Django ORM / SQLAlchemy / Peewee.

```python
Migrator().run()

Customer.find(Customer.last_name == "Brookins").all()
Customer.find(Customer.last_name != "Brookins").all()
Customer.find(Customer.age > 20).all()
Customer.find(Customer.age >= 18).all()
Customer.find(Customer.age < 18).all()
Customer.find(Customer.age <= 50).all()

# AND / OR / NOT
Customer.find((Customer.age > 20) & (Customer.name == "John")).all()
Customer.find((Customer.name == "John") | (Customer.name == "Alice")).all()
Customer.find(~(Customer.age > 30)).all()

# Combined (watch precedence — use parentheses)
Customer.find(
    (Customer.last_name == "Brookins")
    | ((Customer.age == 100) & (Customer.last_name == "Smith"))
).all()
```

### Execute helpers

```python
qs = Customer.find(Customer.age > 18)
qs.all()           # list[Customer]
qs.first()         # one or None/raises per version — check docs
qs.count()
qs.page(offset, limit)
qs.sort_by("age")
qs.sort_by("-age")  # descending
```

### Full-text

```python
bio: Optional[str] = Field(full_text_search=True, default="")

Customer.find(Customer.bio % "python").all()
```

Only fields marked for index / full-text are queryable.

## Embedded models (JsonModel only)

Use **`EmbeddedJsonModel`** for nested documents (not a free-standing top-level collection pattern):

```python
from typing import Optional
from redis_om import EmbeddedJsonModel, JsonModel, Field, Migrator

class Address(EmbeddedJsonModel):
    address_line_1: str
    address_line_2: Optional[str] = None
    city: str = Field(index=True)
    state: str = Field(index=True)
    country: str
    postal_code: str = Field(index=True)

class Customer(JsonModel, index=True):
    first_name: str
    last_name: str
    email: str
    age: int
    bio: Optional[str] = Field(full_text_search=True, default="")
    address: Address

Migrator().run()

Customer.find(
    Customer.address.city == "San Antonio",
    Customer.address.state == "TX",
).all()
```

Lists / optionals on JsonModel:

```python
class User(JsonModel):
    skills: list[str]
    bio: str | None = None
```

## Relationships (manual only)

No `ForeignKey`, `ManyToMany`, or joins.

```python
class Order(JsonModel):
    user_pk: str
    total_cents: int

order = Order.get(oid)
user = User.get(order.user_pk)
```

## Transactions, pipelines, other Redis commands

OM is **not** a transactional SQL ORM. For `WATCH` / `MULTI` / `EXEC`, Lua, Streams, Pub/Sub, cluster ops → **redis-py**.

From a model:

```python
redis_conn = Demo.db()  # connected client
redis_conn.sadd("myset", "a", "b", "c")
```

Or:

```python
from redis_om import get_redis_connection
redis_conn = get_redis_connection()
redis_conn.set("hello", "world")
```

Bulk writes: prefer pipelines over N× `save()` in a tight loop.

## Async (FastAPI / Starlette)

Redis OM Python is **primarily synchronous**. For async apps:

1. Check current package for any async model APIs before betting the architecture on them.
2. Common pattern: `asyncio.to_thread(...)` for OM CRUD/queries, **or** `redis.asyncio` for hot paths and OM for admin/document tooling.
3. Always run `Migrator().run()` off the loop if sync (`await asyncio.to_thread(Migrator().run)`).

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from redis_om import Migrator, JsonModel, Field
import asyncio

class User(JsonModel, index=True):
    name: str = Field(index=True)
    age: int = Field(index=True)

@asynccontextmanager
async def lifespan(app: FastAPI):
    await asyncio.to_thread(Migrator().run)
    yield

app = FastAPI(lifespan=lifespan)

@app.get("/adults")
async def adults():
    return await asyncio.to_thread(lambda: User.find(User.age >= 18).all())
```

## Django integration

- Second model layer **beside** Django ORM — not a replacement
- `Migrator().run()` in `AppConfig.ready()` (guard reloader)
- Relational truth in Django models; OM for Redis-native docs or denormalized read models
- Mirror via services or `post_save` carefully (prefer explicit services)

```python
@receiver(post_save, sender=User)
def mirror_user(sender, instance, **kwargs):
    Customer(pk=str(instance.pk), email=instance.email, name=instance.get_full_name(), age=0).save()
```

## Performance

**Good at:** O(1) PK get, indexed search, validation ergonomics, expressive queries.

**Overhead:** object alloc, Pydantic validation, serialization. Extreme QPS → raw redis-py.

## redis-om vs redis-py vs Django ORM

| Feature | redis-py | Redis OM | Django ORM |
|---|---|---|---|
| Raw commands | Full | Via `.db()` | N/A |
| Pydantic validation | No | Yes | Moderate (forms/serializers) |
| Models | No | Yes | Yes |
| RediSearch | Manual | Automatic indexes | N/A |
| Joins / FK | N/A | No | Yes |
| SQL transactions | N/A | Redis primitives only | Excellent |
| Nested documents | Manual JSON | JsonModel excellent | Poor |
| Speed (KV/doc) | Highest | Slightly lower | DB-bound |

## Schema evolution

- No rich migration framework — plan renames as new field + backfill + drop
- Re-run `Migrator().run()` when indexed fields change (may reindex)
- Version key prefixes on breaking payload changes (`customer:v2:`)

## Testing

- RediSearch queries need **real Redis Stack** (or Enterprise) — use testcontainers; mark integration tests
- fakeredis often insufficient for `find()`

## Best practices

1. Prefer **`JsonModel`** unless Hash is clearly better.
2. **`Field(index=True)` only** on filter/sort/full-text fields.
3. **`Migrator().run()`** on deploy and when indexes change; stay on **DB 0** for search.
4. OM for modeling/CRUD/query; **redis-py** for pipelines, Lua, Pub/Sub, Streams, locks.
5. Relationships = store **`pk` strings**, load explicitly.
6. Set **TTL** (`expire`) on ephemeral docs; memory policy for permanent ones.
7. Paginate `find()` — never unbounded `.all()` on large indexes.
8. Namespace with `global_key_prefix` / separate instances per env.
9. For FastAPI: don't block the event loop on sync OM.
10. Don't treat OM as Postgres substitute for relational integrity.

## Anti-patterns

| Don't | Do |
|---|---|
| Expect joins / select_related | Manual `pk` + second `get` |
| Index every field | Index query surface only |
| Skip Migrator | Run on deploy |
| `find().all()` unbounded | `.page()` / limits |
| OM for multi-row money TX | Postgres + constraints |
| Same prefix prod/stage | Separate prefix or instance |
| Block async loop with `.save()` | `to_thread` or raw async client |

## Minimal end-to-end

```python
from redis_om import JsonModel, Field, Migrator

class User(JsonModel, index=True):
    name: str = Field(index=True)
    age: int = Field(index=True)

Migrator().run()

user = User(name="John", age=30)
user.save()
user.expire(3600)

adults = User.find(User.age >= 18).sort_by("-age").page(0, 25)
```

## Checklist

- [ ] Redis Stack (or JSON+Search) where queries are used
- [ ] Models on DB **0** if using Migrator/indexes
- [ ] Indexes only on needed fields
- [ ] `Migrator().run()` in deploy/startup
- [ ] TTL or memory policy for non-durable data
- [ ] Async boundary respected under FastAPI
- [ ] Clear split: Django/SQLAlchemy ORM vs redis-om responsibilities
- [ ] Fall back to redis-py for advanced Redis features
