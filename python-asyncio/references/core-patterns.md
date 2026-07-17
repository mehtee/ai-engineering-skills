# Core asyncio patterns

## Event loop facts

- Only one thread runs the loop (normally).
- Progress happens when a task hits `await` and yields.
- `async def` functions return **coroutines**; they do nothing until scheduled.
- Prefer **tasks** for concurrent work; bare coroutines are sequential when awaited one-by-one.

```python
# Sequential — total latency = sum
await a(); await b()

# Concurrent — total latency ≈ max (with TaskGroup)
async with asyncio.TaskGroup() as tg:
    ta = tg.create_task(a())
    tb = tg.create_task(b())
# both done here; exceptions surface as ExceptionGroup
```

## Creating concurrency

| API | Use when | Notes |
|---|---|---|
| `asyncio.TaskGroup` | Default for fan-out (3.11+) | Cancels siblings on error; structured |
| `asyncio.gather(*aws, return_exceptions=False)` | Simple gather; older code | First exception cancels? No — others keep running unless you handle |
| `asyncio.create_task` | Fire supervised background **with** strong ownership | Always keep a reference; attach done callback or parent TaskGroup |
| `asyncio.wait` | Need first-completed / custom policies | Lower level |

**Keep a reference** to every task. The event loop does **not** strongly hold tasks forever; orphaned tasks may be GC'd mid-flight and log "Task was destroyed but it is pending".

## Timeouts

```python
# 3.11+ preferred
async with asyncio.timeout(5):
    await fetch()

# portable
await asyncio.wait_for(fetch(), timeout=5)
```

- Put timeouts at **service boundaries** (HTTP client, DB statement, overall request budget).
- Nesting: inner timeout should be shorter than outer when both apply.
- On timeout: `TimeoutError` (3.11 timeout context) / `asyncio.TimeoutError` — treat as expected failure path.

## Cancellation

- Cancel is cooperative: task raises `CancelledError` at the next `await`.
- **Do not** bare `except Exception:` — it swallows cancellation on older patterns; in 3.8+ `CancelledError` bases `BaseException`, but still avoid broad handlers that call `return` without cleanup.
- Always re-raise if you catch `CancelledError` after local cleanup.

```python
async def worker():
    conn = await open_conn()
    try:
        await conn.process()
    finally:
        await conn.close()  # runs on cancel
```

### Shield (use sparingly)

```python
async def save_critical(order):
    # Only shield the commit, not the whole request
    await asyncio.shield(write_to_db(order))
```

Shielding large subgraphs defeats cancellation and causes shutdown hangs. Prefer short critical sections + idempotent writes.

## Semaphores and locks

```python
# Bound fan-out to external API
sem = asyncio.Semaphore(20)

async def call(url: str):
    async with sem:
        return await client.get(url)
```

| Primitive | Purpose |
|---|---|
| `Semaphore(n)` | Limit concurrent entries |
| `Lock` | Mutual exclusion for shared mutable state |
| `Event` | One-shot / multi-wait signal |
| `Condition` | Complex coordination (rare in web apps) |

Prefer **immutable results** + task-local data over heavy locking.

## Queues (pipelines)

```python
async def producer(q: asyncio.Queue):
    for item in source:
        await q.put(item)
    await q.put(None)  # sentinel or use TaskGroup + join pattern

async def consumer(q: asyncio.Queue):
    while True:
        item = await q.get()
        try:
            if item is None:
                break
            await handle(item)
        finally:
            q.task_done()
```

Use queues for **decoupling rates**, not as a substitute for a real job system across processes.

## Exception groups (3.11+)

`TaskGroup` raises `ExceptionGroup` when multiple children fail:

```python
try:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(one())
        tg.create_task(two())
except* ValueError as eg:
    log(eg.exceptions)
```

## contextvars

```python
from contextvars import ContextVar

request_id: ContextVar[str] = ContextVar("request_id", default="-")

async def handle(req):
    token = request_id.set(req.headers["x-request-id"])
    try:
        await business()
    finally:
        request_id.reset(token)
```

Tasks created via `create_task` copy context at creation time — set context **before** spawning children if they must inherit request id.

## Shared resources

```python
# Module or app.state — created once
http: httpx.AsyncClient | None = None

async def startup():
    global http
    http = httpx.AsyncClient(timeout=httpx.Timeout(5.0, connect=2.0), limits=httpx.Limits(max_connections=100))

async def shutdown():
    await http.aclose()
```

## Graceful patterns for request handlers

1. Accept input / validate (sync OK if pure CPU-light).
2. Fan-out I/O under TaskGroup + timeouts + semaphore.
3. Aggregate results; partial failure policy explicit (`return_exceptions` or per-task Output DTO).
4. Persist under a tight critical section.
5. Return response; **do not** start unbounded background work without a queue.

## Minimal "Output" style for partial fan-out

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class CallResult:
    ok: bool
    data: dict | None = None
    error: str = ""

async def guarded(label: str, coro) -> CallResult:
    try:
        async with asyncio.timeout(2):
            return CallResult(ok=True, data=await coro)
    except TimeoutError:
        return CallResult(ok=False, error=f"{label} timeout")
    except Exception as exc:  # narrow in real code
        return CallResult(ok=False, error=f"{label}: {exc}")
```

Use when business wants partial success instead of TaskGroup fail-fast.