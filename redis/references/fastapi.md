# FastAPI + Redis

Prefer **`redis.asyncio`** with a **process-wide pool** created in **lifespan**. Inject via `app.state` or `Depends`. Never open a new client per request.

## Dependencies

```bash
pip install "redis[hiredis]>=5"   # hiredis optional C parser
# or: uv add redis
```

Redis server: 6.2+ recommended; Redis Stack if using redis-om / JSON / Search.

## Lifespan: one async client

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, Request, Depends
from redis.asyncio import Redis
import os

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.redis = Redis.from_url(
        os.environ["REDIS_URL"],  # redis://localhost:6379/0
        encoding="utf-8",
        decode_responses=True,
        max_connections=20,
        socket_connect_timeout=2,
        socket_timeout=2,
        health_check_interval=30,
    )
    # Fail fast at boot (optional)
    await app.state.redis.ping()
    yield
    await app.state.redis.aclose()

app = FastAPI(lifespan=lifespan)

async def get_redis(request: Request) -> Redis:
    return request.app.state.redis
```

**Notes**

- `decode_responses=True` → str in/out; `False` → bytes (better for raw binary/msgpack)
- `max_connections` is **per process**; total ≈ workers × max_connections
- Use separate Redis **logical DBs** or **key prefixes** for cache vs rate-limit vs Celery if sharing an instance (prefixes preferred over many DB numbers in cluster mode)

## Cache-aside endpoint

```python
import json
from fastapi import HTTPException

async def get_product(product_id: str, redis: Redis = Depends(get_redis)):
    key = f"shop:prod:v1:{product_id}"
    cached = await redis.get(key)
    if cached is not None:
        return json.loads(cached)

    product = await db_fetch_product(product_id)  # async DB/httpx
    if product is None:
        raise HTTPException(404)
    await redis.set(key, json.dumps(product), ex=300)
    return product
```

Invalidate on write:

```python
async def update_product(product_id: str, body: ProductIn, redis: Redis = Depends(get_redis)):
    product = await db_update_product(product_id, body)
    await redis.delete(f"shop:prod:v1:{product_id}")
    return product
```

## Stampede control (single-flight lock)

```python
import asyncio
import uuid

async def get_with_lock(redis: Redis, key: str, factory, ttl: int = 300):
    cached = await redis.get(key)
    if cached is not None:
        return json.loads(cached)

    token = str(uuid.uuid4())
    lock_key = f"lock:{key}"
    got = await redis.set(lock_key, token, nx=True, ex=10)
    if got:
        try:
            value = await factory()
            await redis.set(key, json.dumps(value), ex=ttl)
            return value
        finally:
            # release only if we own the lock
            await redis.eval(
                """
                if redis.call('get', KEYS[1]) == ARGV[1] then
                  return redis.call('del', KEYS[1])
                else return 0 end
                """,
                1,
                lock_key,
                token,
            )
    # someone else computing — brief wait then re-read
    await asyncio.sleep(0.05)
    cached = await redis.get(key)
    if cached is not None:
        return json.loads(cached)
    return await factory()  # last resort
```

## Fixed-window rate limit dependency

```python
from fastapi import HTTPException, status

async def rate_limit(
    request: Request,
    redis: Redis = Depends(get_redis),
    limit: int = 60,
    window_sec: int = 60,
):
    # identity: user id if auth else IP
    ident = request.headers.get("x-user-id") or (request.client.host if request.client else "unknown")
    key = f"rl:api:{ident}:{window_sec}"
    try:
        count = await redis.incr(key)
        if count == 1:
            await redis.expire(key, window_sec)
        if count > limit:
            raise HTTPException(
                status_code=status.HTTP_429_TOO_MANY_REQUESTS,
                detail="Rate limit exceeded",
                headers={"Retry-After": str(window_sec)},
            )
    except HTTPException:
        raise
    except Exception:
        # policy: fail open for availability, or fail closed for abuse-sensitive routes
        pass
```

Attach: `dependencies=[Depends(rate_limit)]` on router or route.

## JWT denylist (logout)

```python
async def revoke_jti(redis: Redis, jti: str, exp_remaining_sec: int):
    if exp_remaining_sec > 0:
        await redis.set(f"jwt:deny:{jti}", "1", ex=exp_remaining_sec)

async def assert_not_revoked(redis: Redis, jti: str):
    if await redis.get(f"jwt:deny:{jti}"):
        raise HTTPException(401, "Token revoked")
```

## Distributed lock (simple, single Redis)

```python
from redis.asyncio.lock import Lock

async def transfer(...):
    redis = request.app.state.redis
    async with redis.lock(f"lock:account:{id}", timeout=10, blocking_timeout=5):
        ...  # critical section
```

For money-critical correctness, pair with **DB constraints** / **fencing tokens**; do not rely on lock alone across partitions. See best-practices.

## Pub/Sub (live events)

```python
# publisher
await redis.publish("orders", json.dumps({"id": order_id}))

# subscriber task started in lifespan (one task per process — careful with multi-worker fanout)
async def reader(app: FastAPI):
    pubsub = app.state.redis.pubsub()
    await pubsub.subscribe("orders")
    async for msg in pubsub.listen():
        if msg["type"] == "message":
            await handle(msg["data"])
```

Pub/Sub is **fire-and-forget** (no replay). Need durability → **Streams** (`XADD` / `XREADGROUP`).

## Streams sketch

```python
await redis.xadd("stream:jobs", {"type": "email", "user_id": "1"})
# consumer group workers outside the request path preferred
```

## Sync code inside FastAPI

If a library only has sync `redis.Redis`:

```python
import asyncio
from redis import Redis as SyncRedis

# better: use redis.asyncio
# if stuck:
await asyncio.to_thread(sync_redis.get, key)
```

Do **not** call blocking `sync_redis.get` directly inside `async def`.

## Testing

```python
# pytest + fakeredis (async)
import fakeredis.aioredis

@pytest.fixture
async def redis_client():
    r = fakeredis.aioredis.FakeRedis(decode_responses=True)
    yield r
    await r.aclose()
```

Or testcontainers with real Redis for integration tests.

## Production checklist (FastAPI)

- [ ] `REDIS_URL` from env; TLS (`rediss://`) in prod
- [ ] Lifespan ping + clean `aclose`
- [ ] Timeouts set; pool sized to workers
- [ ] Key prefix includes env (`prod:` / `staging:`)
- [ ] Rate limit and cache failure policies documented
- [ ] Metrics: hit rate, latency, errors, pool in-use
