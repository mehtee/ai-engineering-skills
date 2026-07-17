# Sync / async boundaries

The hard part of asyncio in real systems is **not** syntax — it is crossing libraries and frameworks that disagree on blocking.

## Decision tree

```
Is the call non-blocking / async-native?
  YES → await it on the loop
  NO  → Is it short, pure CPU (< few ms)?
          YES → may run inline (still avoid heavy crypto)
          NO  → Is it I/O or heavy CPU?
                  I/O or uncertain → thread pool (to_thread / sync_to_async)
                  Heavy CPU → ProcessPoolExecutor or separate worker service
```

## `asyncio.to_thread` (stdlib, 3.9+)

```python
result = await asyncio.to_thread(sync_fn, arg1, kw=val)
```

- Uses the default `ThreadPoolExecutor` (size limited — do not flood).
- Good for: sync HTTP SDKs, file I/O, compression, light CPU.
- Not for: Django ORM when `thread_sensitive` semantics matter (use asgiref).

## `asgiref.sync.sync_to_async` (Django world)

```python
from asgiref.sync import sync_to_async

@sync_to_async(thread_sensitive=True)  # default True
def get_user(uid):
    return User.objects.get(pk=uid)

user = await get_user(uid)
# or
user = await sync_to_async(User.objects.get)(pk=uid)
```

| Flag | Meaning |
|---|---|
| `thread_sensitive=True` | Single dedicated thread for Django thread-sensitive state (default ORM safety) |
| `thread_sensitive=False` | Runs in general pool — higher concurrency, unsafe for many Django internals |

**Prefer native async ORM** (`aget`, `aupdate`, …) when available over wrapping every query.

## `async_to_sync`

```python
from asgiref.sync import async_to_sync

def report():
    return async_to_sync(fetch_all_async)()
```

Use from **sync** contexts (management commands, Celery-like workers, sync views). **Never** call from a running event loop thread.

## Dual-mode APIs (library design)

When writing a library consumed by both worlds:

1. Implement **async core** (`async def _fetch()`).
2. Expose sync facade via `async_to_sync` only at the edge **or** document sync as thin wrapper for scripts.
3. Avoid maintaining two divergent business logic copies.

```python
async def fetch_order_async(order_id: str) -> Order:
    ...

def fetch_order(order_id: str) -> Order:
    try:
        asyncio.get_running_loop()
    except RuntimeError:
        return asyncio.run(fetch_order_async(order_id))
    raise RuntimeError("fetch_order() sync API called from async context; use fetch_order_async")
```

Failing loud beats silent deadlock.

## FastAPI threadpool

Sync `def` routes run in a threadpool automatically. That is a valid design:

```python
@app.get("/report")
def report():  # sync — OK for ORM-heavy / sync SDK
    return sync_service.build_report()

@app.get("/prices")
async def prices():  # async — fan-out HTTP
    return await fanout_prices()
```

**Do not** call blocking code inside `async def` hoping FastAPI will move it — it will not.

## Django: async view + sync ORM

```python
from asgiref.sync import sync_to_async

async def order_detail(request, pk):
    order = await Order.objects.select_related("customer").aget(pk=pk)
    # related sync access can re-enter issues — fetch async or serialize carefully
    return JsonResponse({"id": str(order.id), "total": str(order.total)})
```

Avoid:

```python
async def bad(request, pk):
    order = await sync_to_async(Order.objects.get)(pk=pk)
    name = order.customer.name  # implicit sync ORM fetch in async context → SynchronousOnlyOperation
```

Use `select_related` / `prefetch_related` **before** leaving async ORM, or `await sync_to_async(getattr)(...)` only as last resort.

## Transactions

- `transaction.atomic()` is **sync** and thread-sensitive.
- Async path: use async transaction APIs where supported, or wrap the whole atomic block in one `sync_to_async` unit — do not interleave random awaits inside an open sync atomic block across threads.

```python
@sync_to_async
def create_order_sync(dto):
    with transaction.atomic():
        ...
```

## Contextvars vs thread-locals

| | contextvars | threading.local |
|---|---|---|
| Async tasks | Propagates to child tasks | Does **not** cross threads correctly for async |
| Thread pool | Not auto-copied into `to_thread` workers | Works per thread |

When bridging to threads, **pass request id as argument** or re-bind context inside the threaded function.

## Testing boundaries

- Assert that async routes do not call known-blocking APIs (lint, import-linter, custom grep in CI).
- Unit-test sync services as sync; async adapters with `pytest-asyncio`.
- Integration tests should fail if `SynchronousOnlyOperation` appears (Django).

## Practical service layering

```
async route / view
  → async use case (orchestration, TaskGroup)
      → async ports (httpx, async DB)
      → sync ports wrapped once at adapter edge (to_thread / sync_to_async)
```

Keep wraps at **adapters**, not sprinkled inside domain logic.