---
name: python-asyncio
description: >
  Python asyncio best practices for concurrent I/O, with deep guidance for FastAPI
  and Django/DRF. Covers event-loop mental model, TaskGroup, timeouts/cancellation,
  structured concurrency, sync/async boundaries (sync_to_async / threadpool), async
  ORM and HTTP clients, connection pools, testing, and production pitfalls (blocking
  the loop, N+1 awaits, fire-and-forget). Use when writing or reviewing async Python,
  designing async APIs, converting sync views, debugging event-loop stalls, or
  choosing between async and sync stacks. Not for Celery/background job queues or
  multiprocessing CPU work.
---

# Python Asyncio (FastAPI · Django/DRF)

Lead-engineer guidance for **correct, structured, production-grade asyncio** — especially inside web stacks. Prefer **structured concurrency**, **explicit boundaries**, and **I/O-bound concurrency only**.

## Mental model (non-negotiable)

```
One process → one (main) event loop → many Tasks
Tasks interleave only at await points
CPU-bound or blocking I/O on the loop freezes ALL tasks
```

| Work type | Where it runs | Mechanism |
|---|---|---|
| Network / true async I/O | Event loop | `await`, async libs |
| Blocking I/O (sync SDK, ORM without async) | Thread pool | `asyncio.to_thread`, `sync_to_async` |
| CPU-heavy | Process pool / separate workers | `ProcessPoolExecutor`, not asyncio |
| Long background jobs | Task queue / separate service | Out of request path (not covered here) |

**Rule:** If the library has no real non-blocking path, do **not** mark the call site `async` and pretend. Bridge explicitly or stay sync.

## Mode selection

| Signals | Mode |
|---|---|
| New FastAPI service, async endpoints, httpx/asyncpg/redis | **FastAPI build** |
| Django ASGI, async views, async ORM, DRF + async | **Django/DRF build** |
| Core patterns, TaskGroup, cancel, timeouts, anti-patterns | **Core** (always load first if unsure) |
| "Why is my API slow / loop blocked / cancelled error" | **Debug** → anti-patterns + testing refs |
| Convert sync → async | **Boundary** ref + framework ref |

## When to use async (and when not)

**Use async when**
- Many concurrent I/O waits per process (HTTP fan-out, multi-DB, websockets, SSE)
- FastAPI (native async) or Django ASGI with real async backends
- Streaming responses, long-lived connections

**Stay sync when**
- Mostly ORM + CPU with little concurrent I/O
- Critical path depends on sync-only libraries you cannot offload cleanly
- Team cannot yet enforce "no blocking on the loop"
- Django WSGI deployment with no ASGI plan

Premature `async def` everywhere without async I/O is **slower and harder to debug**.

## Instructions

### 1. Analyze

1. **I/O map**: which calls wait on network/disk/DB? Which are CPU?
2. **Stack**: FastAPI / Django ASGI / scripts / workers
3. **Libraries**: async-native vs sync-only
4. **Concurrency shape**: fan-out, pipeline, request-scoped, long-lived task
5. **Failure model**: timeouts, cancellation, partial success

### 2. Design

1. **Structured concurrency** — prefer `asyncio.TaskGroup` (3.11+) over bare `create_task` + gather soup.
2. **Timeouts at boundaries** — every external call has a budget (`asyncio.timeout` / `wait_for`).
3. **Cancellation safety** — `try/finally` for cleanup; shield only the critical commit section, never whole requests.
4. **Pools** — one shared client/pool per process (httpx, DB, Redis); never open a client per request.
5. **Context** — `contextvars` for request id / tenant; do not pass thread-locals across async.

### 3. Implement (core patterns)

**TaskGroup fan-out (preferred)**
```python
import asyncio
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class PriceQuote:
    source: str
    amount: float

async def fetch_quote(source: str, sku: str) -> PriceQuote:
    async with asyncio.timeout(2.0):
        amount = await client.get_price(source, sku)
    return PriceQuote(source=source, amount=amount)

async def best_quote(sku: str, sources: list[str]) -> list[PriceQuote]:
    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(fetch_quote(s, sku)) for s in sources]
    return [t.result() for t in tasks]
```

**Bounded concurrency**
```python
async def map_bounded(coro_fns: list, limit: int = 10) -> list:
    sem = asyncio.Semaphore(limit)
    results: list = [None] * len(coro_fns)

    async def run(i: int, fn) -> None:
        async with sem:
            results[i] = await fn()

    async with asyncio.TaskGroup() as tg:
        for i, fn in enumerate(coro_fns):
            tg.create_task(run(i, fn))
    return results
```

**Sync bridge (blocking library)**
```python
import asyncio

def legacy_sync_sdk(order_id: str) -> dict:
    return SyncClient().fetch(order_id)  # blocks

async def fetch_order(order_id: str) -> dict:
    return await asyncio.to_thread(legacy_sync_sdk, order_id)
```

Django variant: `asgiref.sync.sync_to_async(fn, thread_sensitive=True)` for ORM/thread-sensitive state.

**Never block the loop**
```python
# BAD — freezes every connection on this worker
async def bad(request):
    time.sleep(1)
    rows = connection.cursor().execute(...)  # sync DB on loop
    return rows

# GOOD
async def good(request):
    await asyncio.sleep(1)
    rows = await async_conn.fetch(...)
    return rows
```

### 4. Framework defaults

| Concern | FastAPI | Django / DRF |
|---|---|---|
| Entry | `async def` routes on uvicorn/hypercorn | ASGI (` Daphne`/`uvicorn` + `asgiref`); async views |
| HTTP client | `httpx.AsyncClient` lifespan-shared | same, or `aiohttp` |
| DB | async engine (SQLAlchemy 2 async, asyncpg) | async ORM (`aget`, `afilter`, …) + async backend support |
| Sync ORM/code | `asyncio.to_thread` / run in threadpool | `sync_to_async` / sync views on threadpool |
| Auth/deps | async dependencies | async middleware caveats; prefer async-safe stacks |
| DRF | N/A | Many DRF bits still sync — bridge or thin async views |

Load framework refs for concrete code: `references/fastapi.md`, `references/django-drf.md`.

### 5. Validate (checklist)

**Correctness**
- [ ] No `time.sleep`, sync HTTP, sync disk, or sync DB on the loop
- [ ] Every external I/O has timeout
- [ ] TaskGroup / explicit lifetime for child tasks (no orphan `create_task`)
- [ ] Cancellation / `CancelledError` not swallowed unless re-raised
- [ ] Shared clients created once (lifespan / app state), closed on shutdown

**Performance**
- [ ] Fan-out uses concurrency with a **bound** (semaphore)
- [ ] No sequential `await` in a loop where fan-out is safe
- [ ] Connection pools sized to worker concurrency
- [ ] CPU work not on the loop

**Operability**
- [ ] Request timeouts aligned with gateway / load balancer
- [ ] Structured logging with request id (`contextvars`)
- [ ] Tests use `pytest-asyncio` (or Django async test client) and assert concurrency where claimed

## Bundled resources

Load only what the task needs.

| File | Use for |
|---|---|
| `references/core-patterns.md` | TaskGroup, gather, queues, semaphores, timeouts, cancellation, shields, contextvars |
| `references/anti-patterns.md` | Blocking the loop, fire-and-forget, N+1 awaits, swallowed cancel, unbounded gather |
| `references/sync-async-boundary.md` | `to_thread`, `sync_to_async`, thread-sensitive ORM, dual-mode APIs |
| `references/fastapi.md` | Routes, deps, lifespan clients, BackgroundTasks limits, streaming, websockets |
| `references/django-drf.md` | ASGI, async views, async ORM, DRF limits, signals, middleware |
| `references/testing-production.md` | pytest-asyncio, httpx ASGI transport, diagnostics, uvicorn workers, graceful shutdown |

## Core principles (short)

1. **Structured concurrency** — children outlive nothing; parents own lifetimes (`TaskGroup`).
2. **Never block the loop** — bridge or reject sync I/O in async paths.
3. **Timeouts and cancellation are features** — design for them, not against them.
4. **Bound all concurrency** — semaphores and pool sizes beat unbounded fan-out.
5. **One shared client per resource per process**.
6. **Async is not faster CPU** — it is higher concurrency for waiting.
7. **Explicit > clever** — dual sync/async APIs only when required; prefer one clear mode per service.

## Versions

- Python **3.11+** preferred (`TaskGroup`, `ExceptionGroup`, `asyncio.timeout`)
- Python **3.12+** fine; use modern typing
- FastAPI + anyio/asyncio runtime via uvicorn
- Django **4.2+** / **5.x** for mature async ORM helpers; ASGI deployment required for async views to matter

## Out of scope

- Celery / RQ / ARQ job design (separate concern)
- Multiprocessing numerical compute frameworks
- Trio-only APIs (concepts transfer; examples are asyncio)