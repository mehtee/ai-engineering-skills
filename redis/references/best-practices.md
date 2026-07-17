# Redis best practices (Python APIs)

## Connection management

1. **One pool per process** — FastAPI lifespan / Django cache backend / module singleton.
2. **Timeouts always** — connect + read/write socket timeouts (1–3s typical for app caches).
3. **Health checks** — `health_check_interval` (redis-py) to drop dead connections.
4. **Size the pool** — `workers × max_connections` < Redis `maxclients`; leave headroom for admin/replicas.
5. **TLS in prod** — `rediss://` + valid CA; don't disable cert checks casually.
6. **Close on shutdown** — `aclose()` / pool disconnect to avoid connection leaks on deploy.

## Key design

```
{app}:{env}:{domain}:{entity}:{id}:{field?}
shop:prod:cache:product:42
shop:prod:rl:ip:203.0.113.10
```

- **Short but clear** — keys are stored in every command; avoid 1KB key names
- **Hash tags** for cluster multi-key ops: `{user:42}:profile`, `{user:42}:cart`
- **Version in key** (`v2`) for breaking payload changes
- **No user-controlled raw key segments** without sanitization (key injection / huge cardinality)

## TTL discipline

| Data | TTL guidance |
|---|---|
| Response cache | 30s–15m aligned to staleness budget |
| Session | match auth policy (e.g. 1h–2w) + sliding refresh carefully |
| Rate limit windows | exact window length |
| Locks | short (5–30s) + extend only if safe |
| Idempotency keys | full client retry horizon (e.g. 24h) |
| JWT denylist | remaining token lifetime |

**Immortal keys** only for true reference data with a purge job or `maxmemory-policy=noeviction` and memory alerts.

## Caching patterns

### Cache-aside (lazy) — default

```
read: get → miss → load DB → set TTL
write: write DB → delete key (or set new)
```

### Write-through

```
write: write DB + set cache (same request)
```

Stronger warmth; more write latency.

### Write-behind

Async cache/DB — rare in request/response APIs; complexity high.

### Stampede control

- Mutex lock per key (SET NX) while recomputing  
- **Probabilistic early expiration** (serve stale, recompute in background)  
- **Soft/hard TTL** pair  
- Pre-warm popular keys on deploy  

### Invalidation

- **Delete on write** of that entity  
- **Version counter** for fan-out related keys  
- Prefer delete over updating complex graphs  
- Use `transaction.on_commit` (Django) so failed TX doesn't drop cache early  

## Atomicity

- Single commands are atomic  
- Multi-command: **pipeline** (network batch, not atomic) vs **MULTI/EXEC** (transaction) vs **Lua** (server-side atomic)  
- redis-py:

```python
pipe = redis.pipeline()
pipe.incr(key)
pipe.expire(key, 60)
pipe.execute()
```

```python
redis.eval(lua_script, numkeys, key, arg)
```

## Rate limiting

| Algorithm | Pros | Cons |
|---|---|---|
| Fixed window INCR+EXPIRE | Simple | Boundary burst (2× at edges) |
| Sliding window log (ZSET) | Smooth | More memory per key |
| Sliding window counter | Balance | Slightly more logic |
| Token bucket (Lua) | Bursts allowed | Script maintenance |

Document **fail-open** (availability) vs **fail-closed** (security) when Redis errors.

## Distributed locks

**Minimum viable lock**

```
SET key token NX PX ttl
... critical section ...
DELETE only if token matches (Lua)
```

**Rules**

- Always TTL (process death)  
- Never unlock others' token  
- Keep critical section small  
- Don't assume lock = correctness for money without **DB uniqueness / fencing**  

**Redlock** (multi-node): controversial for correctness; prefer Redis Sentinel/Cluster HA + single-instance lock or a real consensus system for safety-critical work.

Use `redis.lock.Lock` / `redis.asyncio.lock.Lock` rather than hand-rolling when possible.

## Data structures cheat sheet

| Structure | Use |
|---|---|
| STRING | cache blobs, counters, flags |
| HASH | object fields, partial updates |
| LIST | simple queues (no ack) |
| SET | unique membership, tags |
| ZSET | leaderboards, sliding windows |
| STREAM | durable-ish log, consumer groups |
| Pub/Sub | ephemeral notify (no history) |
| JSON (module) | nested docs, redis-om JsonModel |
| Search (module) | secondary index |

## Memory & performance

- **`maxmemory`** + explicit **`maxmemory-policy`** (`allkeys-lru` for pure cache; `noeviction` for queues/locks mixed carefully — **split instances** by workload)  
- Avoid **big keys** (>100KB–1MB values, huge HASH/ZSET) — split or compress  
- Prefer **MGET/pipeline** over chatty round-trips  
- **Connection reuse** > new TCP per command  
- Monitor **evicted_keys**, **blocked_clients**, **instantaneous_ops_per_sec**, slowlog  

## Security

- Prefer **ACL users** with minimal commands/key patterns  
- Bind private network; no public 6379  
- AUTH / TLS  
- Don't store raw passwords; tokenize PII if possible  
- Disable dangerous commands in prod (`FLUSHALL`, `KEYS`, `CONFIG`) via ACL or rename  

## Observability

- Hit ratio = `keyspace_hits / (hits+misses)` (rough; interpret carefully)  
- App-level metrics: cache hit/miss labels, Redis error rate, p99 command latency  
- Alert on memory >80%, continuous evictions for non-cache DBs, connection spikes  

## Anti-patterns

| Anti-pattern | Fix |
|---|---|
| `KEYS *` | `SCAN` cursor loops |
| Unbounded lists/streams | `LTRIM` / `MAXLEN` / TTL policies |
| Pickle untrusted cache | JSON; integrity if needed |
| Cache stampede | lock / single-flight / early expire |
| Shared one Redis DB for Celery + huge cache LRU | separate instances or policies |
| Silent catch-all ignore errors on authz path | fail closed |
| Multi-key ops without hash tags on Cluster | redesign keys |
| Storing multi-MB images | object storage |

## Framework mapping

| Concern | FastAPI | Django/DRF |
|---|---|---|
| Client lifecycle | lifespan + `app.state` | django-redis pool |
| HTTP cache | manual / middleware | `cache_page` / low-level |
| Throttle | Depends + INCR | DRF throttles + shared cache |
| Sessions | custom / starlette session | `SESSION_ENGINE` |
| Models in Redis | redis-om | redis-om beside ORM |
