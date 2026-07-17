# Asyncio anti-patterns

## 1. Blocking the event loop

**Symptoms:** All requests on a worker stall together; latency cliffs; health checks fail under load.

```python
# BAD
async def endpoint():
    time.sleep(0.5)              # blocks loop
    requests.get(url)            # sync HTTP
    open("big.bin").read()       # sync disk
    Model.objects.get(id=1)      # sync ORM on async path (Django)
    hashlib.pbkdf2_hmac(...)     # heavy CPU

# GOOD
async def endpoint():
    await asyncio.sleep(0.5)
    await client.get(url)
    await asyncio.to_thread(Path("big.bin").read_bytes)
    await Model.objects.aget(id=1)
    await asyncio.to_thread(hashlib.pbkdf2_hmac, ...)
```

**Detect:** `asyncio.get_running_loop().slow_callback_duration`; APM span gaps; `py-spy` / `aiomonitor`; log warnings from debug mode (`PYTHONASYNCIODEBUG=1`).

## 2. Sequential awaits that should fan out

```python
# BAD — 3 × RTT
a = await fetch_a()
b = await fetch_b()
c = await fetch_c()

# GOOD
async with asyncio.TaskGroup() as tg:
    ta = tg.create_task(fetch_a())
    tb = tg.create_task(fetch_b())
    tc = tg.create_task(fetch_c())
a, b, c = ta.result(), tb.result(), tc.result()
```

Only keep sequential when later calls need earlier results (true data dependency).

## 3. Unbounded create_task / gather

```python
# BAD — 50k concurrent sockets
await asyncio.gather(*(fetch(u) for u in huge_list))

# GOOD
sem = asyncio.Semaphore(50)
async def bound(u):
    async with sem:
        return await fetch(u)
async with asyncio.TaskGroup() as tg:
    tasks = [tg.create_task(bound(u)) for u in huge_list]
```

Also bound at HTTP/DB pool level; semaphore alone cannot exceed pool capacity usefully.

## 4. Fire-and-forget without ownership

```python
# BAD — task can be GC'd; errors disappear
asyncio.create_task(send_email(user_id))

# BETTER — track + log
background: set[asyncio.Task] = set()

def spawn(coro):
    t = asyncio.create_task(coro)
    background.add(t)
    t.add_done_callback(background.discard)
    t.add_done_callback(lambda task: task.exception() and log.error("bg", exc_info=task.exception()))
    return t
```

For anything business-critical, use a **durable queue**, not in-process tasks (process death loses them). FastAPI `BackgroundTasks` runs after response in the same worker — still not durable.

## 5. Swallowing cancellation

```python
# BAD
async def work():
    try:
        await long_op()
    except Exception:
        return None  # may hide issues; on some stacks cancel handling wrong

# GOOD
async def work():
    try:
        await long_op()
    finally:
        await cleanup()
```

If you catch `asyncio.CancelledError`, re-raise after cleanup.

## 6. Per-request clients and pools

```python
# BAD
async def ep():
    async with httpx.AsyncClient() as client:
        await client.get(url)  # TCP/TLS setup every time

# GOOD — lifespan-scoped client
await request.app.state.http.get(url)
```

Same for DB engines, Redis, gRPC channels.

## 7. Async over sync soup (fake async)

```python
# BAD — async def but all work is sync blocking
async def create_order(data):
    return OrderService().create(data)  # sync ORM/IO

# GOOD options:
# 1) pure sync route/view (threadpool at framework edge)
# 2) real async service stack
# 3) explicit bridge:
async def create_order(data):
    return await asyncio.to_thread(OrderService().create, data)
```

Fake async is worse than honest sync: it signals concurrency you do not have.

## 8. Ignoring ExceptionGroup

```python
# BAD
try:
    async with asyncio.TaskGroup() as tg:
        ...
except Exception as e:  # misses ExceptionGroup structure
    log(e)
```

Use `except*` or inspect `ExceptionGroup.exceptions`.

## 9. Race on shared mutable state

```python
# BAD
total = 0
async def add(n):
    global total
    cur = total
    await asyncio.sleep(0)  # yield — race
    total = cur + n
```

Prefer return values / reduce; if shared state is required, use `asyncio.Lock` or queues.

## 10. Wrong timeout layering

- Gateway 30s, app 60s → worker works after client is gone.
- No timeout on outbound HTTP → hung sockets exhaust pool.

Align: `client_timeout < app_timeout < upstream_timeout` budgets, leave slack for cleanup.

## 11. `asgiref.async_to_sync` inside the loop

Calling `async_to_sync` from code already under a running loop deadlocks or errors. From async code, `await` the coroutine; from sync-only libraries, design dual entrypoints carefully (see boundary ref).

## 12. N+1 awaits (async cousin of N+1 queries)

```python
# BAD
for id in ids:
    users.append(await repo.aget(id))

# GOOD
users = await repo.aget_many(ids)  # one query / one bulk API
```

Concurrency can hide N+1 cost until traffic spikes — fix data access shape first, then concurrent fan-out.

## Review checklist (PR)

- [ ] Any sync I/O/CPU in `async def` paths?
- [ ] Fan-out bounded?
- [ ] Timeouts on outbound calls?
- [ ] Tasks owned (TaskGroup or tracked set)?
- [ ] Clients/pools singleton per process?
- [ ] Cancellation-safe `finally`?
- [ ] No critical durability via fire-and-forget?