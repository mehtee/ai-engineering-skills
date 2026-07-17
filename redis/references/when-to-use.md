# When to use Redis

Decision guide for Python APIs (FastAPI, Django/DRF). Prefer the **simplest** layer that meets latency, sharing, and durability needs.

## Decision tree

```
Need shared state across processes/instances?
  NO  → in-process (functools.lru_cache, cachetools) is enough
  YES → Need durability / complex queries / relations?
          YES → primary DB (Postgres/MySQL); Redis only as cache in front
          NO  → Is it a job that must not be lost?
                  YES → queue/stream with consumer groups (Redis Streams, SQS, Rabbit, Celery broker)
                  NO  → Redis fits (cache, session, limit, lock, pub/sub, ephemeral tokens)
```

## Fit matrix

| Use case | Redis? | Structure / notes | Prefer instead when… |
|---|---|---|---|
| Hot query / rendered response cache | **Yes** | STRING/JSON + TTL; cache-aside | Dataset tiny and single process |
| Session store (multi-instance) | **Yes** | HASH/STRING + TTL; sticky sessions optional | Single instance + signed cookie sessions OK |
| Rate limiting / quotas | **Yes** | INCR+EXPIRE, ZSET sliding, token bucket Lua | Single instance → middleware memory OK |
| Distributed lock / leader | **Yes (careful)** | SET NX PX + token; short TTL | Need strong consensus → etcd/ZooKeeper |
| Idempotency keys (payments) | **Yes** | SET NX + TTL matching retry window | Must be audit-durable → DB unique constraint |
| Feature flags (simple) | **Yes** | HASH / JSON; pub/sub to invalidate local | Complex targeting → dedicated flag service |
| Real-time notifications | **Yes** | Pub/Sub (ephemeral) or Streams | Cross-region durable → Kafka/NATS |
| Task queue | **Sometimes** | LIST or Streams; Celery+Redis broker | Need priority/delay/visibility → RQ/Celery/Arq carefully designed |
| Full-text search over Redis docs | **Yes + Stack** | RediSearch via redis-om | Heavy analytics → OpenSearch/ES/Postgres |
| Primary user/order data | **Rarely** | Only if designed for Redis-as-DB | Default to RDBMS |
| File / media storage | **No** | — | S3/GCS + CDN |
| Cross-row ACID business txns | **No** | — | Postgres |

## Cache vs primary store

**Cache (default)**

- Source of truth is DB or upstream API
- Redis miss → recompute → set with TTL
- Redis loss = temporary load spike, not data loss
- Invalidation on write (delete key or version bump)

**Primary / system-of-record in Redis**

- Only when access patterns are key-based, TTLs natural, and rebuild from elsewhere is acceptable **or** Redis persistence (AOF/RDB) is part of the durability plan
- redis-om / RedisJSON apps sometimes do this for catalogs, leaderboards, presence
- Document backup, eviction policy (`noeviction` vs `allkeys-lru`), and failover

## FastAPI-specific triggers

Use Redis when the ASGI fleet is **>1 worker/replica** and you need:

- Shared rate limits (in-memory `slowapi` is per-process — wrong under multi-worker)
- Shared session/token blacklist (JWT logout denylist)
- Response caching for expensive fan-out aggregations
- WebSocket fan-out across workers (Pub/Sub or Streams; one subscriber process pattern)

Stay without Redis when:

- Single Uvicorn worker prototype
- Pure JWT stateless auth with no denylist
- DB queries already < few ms and QPS is low

## Django/DRF-specific triggers

Use Redis when:

- `LocMemCache` is wrong (multi-process gunicorn/uwsgi, multiple hosts)
- Session affinity is unavailable and sessions must be shared → `SESSION_ENGINE` with cache or signed cookies
- DRF throttling across workers (`AnonRateThrottle` default cache must be shared)
- Per-view or per-queryset caching (`cache_page`, low-level `cache`)
- Celery broker/result backend (common; separate DB index from cache ideally)

Stay without Redis when:

- `DummyCache` in tests / local
- Database cache table is acceptable for low traffic
- Only one process and `LocMemCache` is intentional

## Consistency & staleness budget

| Budget | Pattern |
|---|---|
| 0 stale allowed | Don't cache; or transactional outbox + synchronous invalidate + short TTL safety net |
| Seconds–minutes OK | Cache-aside + TTL; invalidate on write best-effort |
| Minutes–hours OK | Longer TTL; scheduled warmers; versioned keys |
| Eventual across regions | Local cache + Redis + pub/sub invalidation; accept races |

## Cost & complexity signals **against** Redis

- Team has no runbook for memory eviction / hot keys / failover
- Keyspace design unclear → will grow without TTL
- Compliance requires all personal data only in audited DB
- Latency dominated by app logic, not I/O

## Quick recommendations

1. **Default web API**: Postgres + Redis **cache + rate limit + optional sessions**
2. **Stateless JWT API**: Redis for **rate limit + optional denylist** only
3. **Real-time product**: Redis **Pub/Sub or Streams** + Postgres truth
4. **Document-ish Redis models**: **redis-om** only if Redis Stack is available and key access fits
5. **Never**: Redis as the only place a financial ledger lives without a durable design review
