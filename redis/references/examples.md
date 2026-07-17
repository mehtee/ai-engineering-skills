# Examples

Copy-adapt snippets. Adjust key prefixes, TTLs, and failure policy per service.

## 1. FastAPI: product cache + invalidate

```python
# app/redis_client.py
from redis.asyncio import Redis
from functools import lru_cache
import os

def create_redis() -> Redis:
    return Redis.from_url(
        os.environ.get("REDIS_URL", "redis://localhost:6379/0"),
        decode_responses=True,
        max_connections=20,
        socket_connect_timeout=2,
        socket_timeout=2,
    )
```

```python
# app/main.py
import json
from contextlib import asynccontextmanager
from fastapi import FastAPI, Depends, HTTPException
from redis.asyncio import Redis
from app.redis_client import create_redis

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.redis = create_redis()
    await app.state.redis.ping()
    yield
    await app.state.redis.aclose()

app = FastAPI(lifespan=lifespan)

from fastapi import Request

async def redis_dep(request: Request) -> Redis:
    return request.app.state.redis

async def fetch_product_from_db(pid: str) -> dict | None:
    ...

@app.get("/products/{pid}")
async def get_product(pid: str, redis: Redis = Depends(redis_dep)):
    key = f"shop:prod:v1:{pid}"
    raw = await redis.get(key)
    if raw:
        return json.loads(raw)
    product = await fetch_product_from_db(pid)
    if not product:
        raise HTTPException(404, "Not found")
    await redis.set(key, json.dumps(product), ex=300)
    return product

@app.put("/products/{pid}")
async def put_product(pid: str, body: dict, redis: Redis = Depends(redis_dep)):
    product = await save_product_db(pid, body)
    await redis.delete(f"shop:prod:v1:{pid}")
    return product
```

## 2. FastAPI: sliding-ish fixed window rate limit

```python
from fastapi import Request, HTTPException, Depends
from redis.asyncio import Redis

async def limit_user(request: Request, redis: Redis = Depends(redis_dep)):
    user = request.headers.get("x-api-key") or request.client.host
    key = f"rl:v1:{user}"
    pipe = redis.pipeline()
    pipe.incr(key)
    pipe.expire(key, 60, nx=True)  # redis-py 5+: expire nx if supported; else expire only when count==1
    count, _ = await pipe.execute()
    # portable variant:
    # count = await redis.incr(key)
    # if count == 1: await redis.expire(key, 60)
    if int(count) > 100:
        raise HTTPException(429, "Too many requests")
```

## 3. FastAPI: idempotency key for POST

```python
import json
from fastapi import Header, HTTPException

@app.post("/charges")
async def charge(
    body: dict,
    redis: Redis = Depends(redis_dep),
    idempotency_key: str | None = Header(default=None, alias="Idempotency-Key"),
):
    if not idempotency_key:
        raise HTTPException(400, "Idempotency-Key required")
    key = f"idem:charge:{idempotency_key}"
    cached = await redis.get(key)
    if cached:
        return json.loads(cached)

    # SET NX reservation to block concurrent duplicates
    reserved = await redis.set(f"{key}:lock", "1", nx=True, ex=60)
    if not reserved:
        raise HTTPException(409, "Request in progress")

    result = await provider_charge(body)
    await redis.set(key, json.dumps(result), ex=86400)
    await redis.delete(f"{key}:lock")
    return result
```

## 4. Django: cache_page + low-level + on_commit

```python
# views.py
from django.core.cache import cache
from django.db import transaction
from django.views.decorators.cache import cache_page
from rest_framework.decorators import api_view
from rest_framework.response import Response

@cache_page(60)
def public_homepage_stats(request):
    return JsonResponse({"ok": True})

@api_view(["GET"])
def product_detail(request, pk: int):
    key = f"prod:{pk}"
    data = cache.get(key)
    if data is None:
        data = serialize(Product.objects.get(pk=pk))
        cache.set(key, data, timeout=300)
    return Response(data)

@api_view(["PATCH"])
def product_update(request, pk: int):
    with transaction.atomic():
        p = Product.objects.select_for_update().get(pk=pk)
        ...
        p.save()
        transaction.on_commit(lambda: cache.delete(f"prod:{pk}"))
    return Response(serialize(p))
```

## 5. Django: raw atomic counter via django-redis

```python
from django_redis import get_redis_connection

def track_login(user_id: int) -> int:
    r = get_redis_connection("default")
    key = f"login:count:{user_id}"
    n = r.incr(key)
    if n == 1:
        r.expire(key, 86400)
    return n
```

## 6. DRF throttle with shared Redis

```python
# settings.py
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379/2",
        "OPTIONS": {"CLIENT_CLASS": "django_redis.client.DefaultClient"},
        "KEY_PREFIX": "myapp",
    }
}

REST_FRAMEWORK = {
    "DEFAULT_THROTTLE_RATES": {
        "user": "1000/day",
        "anon": "100/day",
        "login": "5/min",
    }
}
```

```python
# throttles.py
from rest_framework.throttling import SimpleRateThrottle

class LoginThrottle(SimpleRateThrottle):
    scope = "login"

    def get_cache_key(self, request, view):
        email = request.data.get("email") or self.get_ident(request)
        return f"throttle_login_{email}"
```

## 7. redis-om JsonModel + embedded (Redis Stack)

```python
from typing import Optional
from redis_om import JsonModel, EmbeddedJsonModel, Field, Migrator

class Address(EmbeddedJsonModel):
    city: str = Field(index=True)
    country: str = Field(index=True)

class Sku(JsonModel):
    code: str = Field(index=True)
    title: str = Field(index=True, full_text_search=True)
    price_cents: int = Field(index=True)
    active: bool = Field(index=True)
    warehouse: Optional[Address] = None

    class Meta:
        global_key_prefix = "shop"
        model_key_prefix = "sku"

# once at startup (Redis logical DB 0 for indexes)
Migrator().run()

sku = Sku(
    code="ABC",
    title="Widget",
    price_cents=999,
    active=True,
    warehouse=Address(city="Berlin", country="DE"),
)
sku.save()
sku.expire(3600)

cheap = (
    Sku.find((Sku.price_cents < 1000) & (Sku.active == True))
    .sort_by("price_cents")
    .page(0, 50)
)
# or: Sku.find(Sku.warehouse.city == "Berlin").all()
```

## 8. Distributed lock with token (sync redis-py)

```python
import uuid
import time
from redis import Redis

RELEASE = """
if redis.call("get", KEYS[1]) == ARGV[1] then
  return redis.call("del", KEYS[1])
else
  return 0
end
"""

def with_lock(r: Redis, name: str, ttl_ms: int = 10000):
    token = str(uuid.uuid4())
    key = f"lock:{name}"
    if not r.set(key, token, nx=True, px=ttl_ms):
        raise RuntimeError("busy")
    try:
        yield
    finally:
        r.eval(RELEASE, 1, key, token)
```

## 9. Pub/Sub fan-out sketch (async)

```python
import asyncio
import json
from redis.asyncio import Redis

async def publish_order(r: Redis, order: dict):
    await r.publish("channel:orders", json.dumps(order))

async def subscribe_orders(r: Redis):
    pubsub = r.pubsub()
    await pubsub.subscribe("channel:orders")
    async for message in pubsub.listen():
        if message["type"] != "message":
            continue
        order = json.loads(message["data"])
        await notify_websockets(order)
```

## 10. Failure policy helper

```python
import logging
from typing import Callable, TypeVar

T = TypeVar("T")
log = logging.getLogger(__name__)

async def cache_get_or(
    redis,
    key: str,
    factory: Callable[[], T],
    ttl: int,
    *,
    fail_open: bool = True,
) -> T:
    try:
        raw = await redis.get(key)
        if raw is not None:
            return json.loads(raw)
    except Exception:
        log.exception("redis get failed key=%s", key)
        if not fail_open:
            raise
        return await factory()

    value = await factory()
    try:
        await redis.set(key, json.dumps(value), ex=ttl)
    except Exception:
        log.exception("redis set failed key=%s", key)
    return value
```

## Env templates

```bash
# FastAPI / generic
REDIS_URL=redis://:password@redis:6379/0

# Django
# CACHE on /1, Celery broker /0, results /2
REDIS_CACHE_URL=redis://redis:6379/1
CELERY_BROKER_URL=redis://redis:6379/0
CELERY_RESULT_BACKEND=redis://redis:6379/2
REDIS_OM_URL=redis://redis:6379/3
```
