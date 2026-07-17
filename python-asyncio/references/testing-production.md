# Testing and production

## pytest-asyncio

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
asyncio_default_fixture_loop_scope = "function"
```

```python
import pytest
import asyncio

@pytest.mark.asyncio
async def test_fanout(httpx_mock):
    ...
    result = await best_quote("sku", ["a", "b"])
    assert len(result) == 2
```

### Principles

- **Await the unit under test**; do not `asyncio.run` inside an already-async test.
- Prefer **fake async servers / respx / httpx mock** over real network.
- Assert **concurrency** when it is the point (e.g. total time &lt; sum of sleeps).
- Assert **cancellation**: cancel task mid-flight, expect cleanup.

```python
@pytest.mark.asyncio
async def test_cancel_closes_connection():
    started = asyncio.Event()
    closed = asyncio.Event()

    async def work():
        started.set()
        try:
            await asyncio.sleep(10)
        finally:
            closed.set()

    t = asyncio.create_task(work())
    await started.wait()
    t.cancel()
    with pytest.raises(asyncio.CancelledError):
        await t
    assert closed.is_set()
```

## FastAPI tests

```python
from httpx import AsyncClient, ASGITransport

@pytest.mark.asyncio
async def test_health(app):
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        r = await ac.get("/health")
    assert r.status_code == 200
```

Use lifespan-aware client setup so `app.state` clients exist.

## Django async tests

```python
from django.test import AsyncClient

@pytest.mark.asyncio
async def test_order_list():
    client = AsyncClient()
    response = await client.get("/api/orders/")
    assert response.status_code == 200
```

Also run under ASGI-like settings; watch for `SynchronousOnlyOperation` in logs.

## Load and diagnostics

| Tool | Use |
|---|---|
| `PYTHONASYNCIODEBUG=1` | Slow callback warnings |
| `loop.slow_callback_duration = 0.05` | Tune threshold |
| `aiomonitor` / `aiodebug` | Live task inspection |
| `py-spy` / `tracemalloc` | Blocking / memory |
| OpenTelemetry async instrumentation | Trace fan-out |

**Smell:** task count grows without bound → leak of create_task / queue consumers.

## Production configuration

### Workers and pools

```
max_db_connections >= (uvicorn_workers * pool_size_per_worker) + margin
```

Same for Redis and upstream HTTP `max_connections`.

### Timeouts (align layers)

```
Load balancer 60s
  → Uvicorn keep-alive / proxy 55s
    → Route budget 50s
      → Outbound HTTP 2–5s each
```

### Graceful shutdown

1. Stop accepting new connections.
2. Allow in-flight requests to finish (bounded drain period).
3. Lifespan: close HTTP clients, DB engines, queue consumers.
4. Cancel remaining tasks; do not shield entire shutdown.

### Logging

```python
import logging
from contextvars import ContextVar

request_id: ContextVar[str] = ContextVar("request_id", default="-")

class RequestIdFilter(logging.Filter):
    def filter(self, record):
        record.request_id = request_id.get()
        return True
```

Include `request_id` in every log line for correlating concurrent tasks.

### Health checks

- `/health/live` — process up (no deps).
- `/health/ready` — can reach DB/Redis (with short timeouts).
- Ready checks must not block the loop (async or short to_thread).

## Common production failures

| Failure | Cause | Fix |
|---|---|---|
| Latency spikes all requests | Blocking call on loop | Find sync I/O/CPU; bridge or remove |
| `Too many connections` | workers × pool | Resize pools; PgBouncer |
| Hung deploys | non-cancellable tasks / shields | shrink shields; handle SIGTERM |
| Silent data loss | fire-and-forget tasks | durable queue |
| Random `CancelledError` logs | client disconnect | expected; clean finally, lower log level if noisy |
| `SynchronousOnlyOperation` | lazy ORM in async | select_related / async APIs |

## Review gates before ship

- [ ] Load test with realistic fan-out
- [ ] Chaos: kill upstream mid-request (timeouts behave)
- [ ] Deploy drain works under traffic
- [ ] Metrics: event loop lag, open connections, task count
- [ ] Error budget on partial fan-out documented