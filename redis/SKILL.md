---
name: redis
description: >
  Redis with Python web stacks: when to use (cache, sessions, rate limits, locks,
  queues, pub/sub, real-time), redis-py / redis.asyncio, django-redis, FastAPI
  lifespan pools, and Redis OM (redis-om HashModel/JsonModel). Use when adding
  Redis to FastAPI or Django/DRF, choosing cache keys/TTLs, fixing stampedes,
  designing rate limiters or distributed locks, modeling data with redis-om, or
  reviewing production Redis settings. Not for Redis Cluster ops deep-dives or
  non-Python clients.
---

# Redis (FastAPI · Django/DRF · redis-om)

Lead-engineer guidance for **correct Redis usage in Python APIs**: pick the right
pattern, one shared client per process, explicit TTLs, and failure modes that do
not take down the request path.

## Mental model

```
Redis = in-memory data structure server (not "just a cache")
One process → one (or few) shared connection pools → many commands
Every key needs a reason to expire or a documented permanent lifecycle
Network + serialization cost still exists — Redis is not free
```

| Work type | Redis role | Typical structures |
|---|---|---|
| Read-heavy hot data | Cache (aside / write-through) | STRING, HASH, JSON |
| Login / cart / CSRF | Session / short-lived state | STRING / HASH + TTL |
| API abuse control | Rate limit / quota | STRING INCR, ZSET, TOKEN bucket |
| Multi-instance mutual exclusion | Distributed lock | SET NX PX + token |
| Background work handoff | Queue / stream | LIST, STREAM |
| Live fan-out | Pub/Sub or Streams | PUBLISH/SUBSCRIBE, XADD |
| Secondary index over Redis docs | redis-om models | HASH / JSON + RediSearch |

**Rule:** If the data is authoritative business state that must survive Redis loss
without rebuild, store it in the primary DB first; Redis is usually **derived**
or **ephemeral** unless you deliberately design Redis-as-primary (rare).

## Mode selection

| Signals | Mode / refs |
|---|---|
| "Should we use Redis for X?" | **when-to-use** |
| FastAPI endpoints, lifespan, `redis.asyncio` | **fastapi** |
| Django cache, sessions, DRF throttling | **django-drf** |
| HashModel / JsonModel / Migrator | **redis-om** |
| Keys, TTL, stampede, locks, pipelines | **best-practices** |
| Copy-paste patterns | **examples** |

## When to use Redis (summary)

**Use Redis when**

- Same response/query is hit often and is expensive (DB, HTTP, compute)
- Cross-instance shared state (sessions, rate limits, feature flags)
- Sub-ms coordination (locks, leader election lite, idempotency keys)
- Real-time or near-real-time fan-out inside your infra
- Short-lived tokens, OTPs, magic links with natural TTL

**Do not use Redis when**

- Data must be strongly consistent and durable as source of truth (use Postgres)
- Traffic is low and DB is already fast enough (complexity tax)
- You need complex multi-row transactions / joins (RDBMS)
- Payload is huge binary blobs (object storage + CDN)
- "We might need it later" with no concrete access pattern

Full decision matrix: [references/when-to-use.md](references/when-to-use.md)

## Instructions

### 1. Analyze

1. **Access pattern**: read-heavy vs write-heavy; key cardinality; value size
2. **Consistency**: stale OK? how long? who invalidates?
3. **Failure**: if Redis is down, fail open, fail closed, or degrade?
4. **Stack**: FastAPI async vs Django sync WSGI vs Django ASGI
5. **Multi-tenant**: key namespace (`app:env:tenant:…`)

### 2. Design

1. **One pool per process** — create in lifespan / `AppConfig.ready` / module init carefully; never per-request `Redis()`.
2. **Key schema** — `{app}:{env}:{domain}:{id}:{variant}` + documented TTLs.
3. **TTL on every ephemeral key** — no immortal cache keys by accident.
4. **Serialize deliberately** — JSON for portability; msgpack/orjson when hot; never pickle untrusted data.
5. **Timeouts** — socket connect/read timeouts so Redis blips do not hang workers.
6. **Idempotency & locks** — token-based unlock; short TTL; no infinite hold.

### 3. Implement by stack

- **FastAPI** → [references/fastapi.md](references/fastapi.md) (`redis.asyncio`, lifespan, Depends)
- **Django / DRF** → [references/django-drf.md](references/django-drf.md) (django-redis, cache framework, sessions, throttles)
- **redis-om** → [references/redis-om.md](references/redis-om.md) (models, indexes, async)

### 4. Harden

Load [references/best-practices.md](references/best-practices.md):

- Cache-aside + stampede control (lock / probabilistic early expire)
- Pipeline / transaction (`MULTI/EXEC`) for multi-key atomicity where needed
- Rate limit algorithms (fixed window vs sliding vs token bucket)
- Distributed lock pitfalls (Redlock caveats; prefer single-instance lock + fencing token for correctness-critical paths)
- Memory: `maxmemory-policy`, key size, big-key avoidance
- Observability: command latency, hit ratio, evictions, connected clients

### 5. Validate (checklist)

- [ ] Shared client/pool; closed on shutdown
- [ ] TTLs documented; no unbounded key growth
- [ ] Timeouts + error handling (degrade path tested)
- [ ] Keys namespaced; no cross-env collisions
- [ ] Cache invalidation path exists for writes
- [ ] Rate limits / locks use correct atomic commands
- [ ] Secrets not stored in Redis without encryption policy
- [ ] Load test: connection count = workers × pool size within Redis `maxclients`

## Bundled resources

| File | Load when |
|---|---|
| [references/when-to-use.md](references/when-to-use.md) | Choosing Redis vs DB vs in-process vs queue |
| [references/fastapi.md](references/fastapi.md) | FastAPI / Starlette / async workers |
| [references/django-drf.md](references/django-drf.md) | Django cache, sessions, DRF |
| [references/redis-om.md](references/redis-om.md) | ORM-style models on Redis |
| [references/best-practices.md](references/best-practices.md) | Production patterns & anti-patterns |
| [references/examples.md](references/examples.md) | End-to-end snippets |

## Anti-patterns (quick)

| Don't | Do |
|---|---|
| `Redis()` inside every request | One pool; inject client |
| Cache without TTL | TTL + explicit permanent keys only |
| `KEYS *` in production | `SCAN` / application indexes |
| Pickle application cache | JSON / msgpack; signed if needed |
| Ignore Redis errors in rate limiter | Define fail-open vs fail-closed |
| Use Pub/Sub as durable queue | Streams or a real broker |
| Store multi-MB values | Split, compress, or use object store |
| Unlock lock without token check | Compare-and-delete Lua / library |

## Package map

| Need | Package |
|---|---|
| Sync Redis client | `redis` (`redis.Redis`) |
| Async Redis client | `redis` (`redis.asyncio`) |
| Django cache backend | `django-redis` |
| ORM-ish models + secondary index | `redis-om` (Pydantic + Hash/JSON; Redis Stack for `find()` / indexes; primarily sync) |
| FastAPI rate limit helpers | often custom or `slowapi` (in-memory) — prefer Redis-backed custom for multi-instance |

## Related skills

- **python-asyncio** — event loop, pools, no blocking on the loop
- **django-clean-drf** — where cache sits in Clean Architecture (infrastructure layer)
