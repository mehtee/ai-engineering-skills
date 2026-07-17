# Django / DRF + Redis

Use Redis as a **shared cache**, **session backend**, **throttle storage**, and optionally **Celery broker**. Prefer **django-redis** over the stock locmem/DB cache for multi-process deployments.

## Install

```bash
pip install django-redis redis
```

## Settings: cache

```python
# settings.py
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": env("REDIS_URL", default="redis://127.0.0.1:6379/1"),
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
            "SOCKET_CONNECT_TIMEOUT": 2,
            "SOCKET_TIMEOUT": 2,
            "CONNECTION_POOL_KWARGS": {"max_connections": 50},
            "IGNORE_EXCEPTIONS": True,  # cache blip ≠ take down site (see tradeoffs)
            # "PARSER_CLASS": "redis.connection.HiredisParser",  # if hiredis installed
        },
        "KEY_PREFIX": "myapp",
        "TIMEOUT": 300,  # default TTL seconds
    }
}
```

**`IGNORE_EXCEPTIONS`**

- `True` (common with django-redis + middleware): fail open — site works without cache
- `False`: errors propagate — use when cache is required for correctness (rare)

Also set:

```python
DJANGO_REDIS_LOG_IGNORED_EXCEPTIONS = True
```

## Sessions

**Option A — cache sessions** (fast; sessions lost if Redis flushes unless you accept that):

```python
SESSION_ENGINE = "django.contrib.sessions.backends.cache"
SESSION_CACHE_ALIAS = "default"
```

**Option B — cached_db** (write-through DB; Redis accelerates reads):

```python
SESSION_ENGINE = "django.contrib.sessions.backends.cached_db"
```

**Option C — signed cookies** (no Redis; size limits; not for sensitive server-side state).

For multi-server **use A or B with shared Redis**, not `LocMemCache`.

## Low-level cache API

```python
from django.core.cache import cache

def get_profile(user_id: int) -> dict:
    key = f"profile:{user_id}"
    data = cache.get(key)
    if data is not None:
        return data
    data = expensive_load(user_id)
    cache.set(key, data, timeout=600)
    return data

def bump_profile(user_id: int) -> None:
    cache.delete(f"profile:{user_id}")
```

Versioned keys (avoid mass delete):

```python
def profile_key(user_id: int) -> str:
    v = cache.get(f"profile:ver:{user_id}") or 1
    return f"profile:v{v}:{user_id}"

def invalidate_profile(user_id: int) -> None:
    key = f"profile:ver:{user_id}"
    try:
        cache.incr(key)
    except ValueError:
        cache.set(key, 2, timeout=None)
```

## View / page cache

```python
from django.views.decorators.cache import cache_page

@cache_page(60 * 5)
def public_stats(request):
    ...
```

Prefer **per-entity low-level cache** over `cache_page` for authenticated or highly personalized DRF APIs.

## DRF throttling (shared)

DRF stores throttle counters in the **default cache**. With LocMemCache, each Gunicorn worker has its own counters → users get N× limit. **django-redis fixes this.**

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.AnonRateThrottle",
        "rest_framework.throttling.UserRateThrottle",
    ],
    "DEFAULT_THROTTLE_RATES": {
        "anon": "30/min",
        "user": "120/min",
        "burst": "10/sec",
    },
}
```

Custom scope:

```python
from rest_framework.throttling import SimpleRateThrottle

class BurstThrottle(SimpleRateThrottle):
    scope = "burst"

    def get_cache_key(self, request, view):
        if request.user.is_authenticated:
            ident = request.user.pk
        else:
            ident = self.get_ident(request)
        return self.cache_format % {"scope": self.scope, "ident": ident}
```

## Service-layer cache (Clean Architecture)

Keep Redis in **infrastructure**; domain services accept a port or use Django cache at the edge:

```python
# infrastructure/cache.py
from django.core.cache import cache

class RedisProductCache:
    def get(self, product_id: str):
        return cache.get(f"prod:{product_id}")

    def set(self, product_id: str, payload: dict, ttl: int = 300):
        cache.set(f"prod:{product_id}", payload, timeout=ttl)

    def invalidate(self, product_id: str):
        cache.delete(f"prod:{product_id}")
```

Use cases call the port; do not scatter raw Redis keys in views.

## Async Django (ASGI)

Django 4.1+ async views: `cache` API is **sync**. Bridge explicitly:

```python
from asgiref.sync import sync_to_async
from django.core.cache import cache

data = await sync_to_async(cache.get)(key)
await sync_to_async(cache.set)(key, value, 300)
```

Or use `redis.asyncio` client managed yourself for hot async paths (parallel to django-redis), same rules as FastAPI lifespan — typically in ASGI middleware or a shared module with lazy init. Avoid double pools without reason.

## Raw redis client from django-redis

```python
from django_redis import get_redis_connection

conn = get_redis_connection("default")  # sync redis.Redis
conn.incr("metrics:signups")
```

Use for atomic ops not exposed by the cache API (INCR, Lua, SET NX).

## Select for update + cache

Never use cache as a substitute for `select_for_update()` on financial rows. Pattern:

1. DB transaction + row lock for correctness  
2. Cache invalidate after commit  

```python
from django.db import transaction

@transaction.atomic
def apply_order(order_id):
    order = Order.objects.select_for_update().get(pk=order_id)
    ...
    transaction.on_commit(lambda: cache.delete(f"order:{order_id}"))
```

## Celery + Redis

Common: Redis as broker **and** result backend. **Isolate**:

- Logical DB `/0` broker, `/1` results, `/2` django-redis cache — **or** separate instances  
- Key prefixes if same DB  

```python
CELERY_BROKER_URL = "redis://127.0.0.1:6379/0"
CELERY_RESULT_BACKEND = "redis://127.0.0.1:6379/1"
```

Do not use the same maxmemory-allkeys-lru instance for broker without monitoring — eviction can drop jobs.

## Tests

```python
# settings/test.py
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.locmem.LocMemCache",
    }
}
```

Or `fakeredis` with a custom backend for closer behavior. Always isolate throttle tests from shared Redis.

## Production checklist (Django)

- [ ] `KEY_PREFIX` + different Redis DB/instance per env
- [ ] Shared cache for all app workers (no LocMem in prod)
- [ ] Session engine matches multi-instance topology
- [ ] DRF throttles verified under 2+ workers
- [ ] `on_commit` invalidation for write paths
- [ ] Celery not fighting cache for memory
- [ ] Timeouts + optional `IGNORE_EXCEPTIONS` intentional
