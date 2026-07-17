# Django / DRF + asyncio

Django supports async views, middleware (partial), and async ORM operations under **ASGI**. Async is opt-in per call path; much of the ecosystem remains sync.

## Deploy ASGI

```python
# asgi.py
from django.core.asgi import get_asgi_application
application = get_asgi_application()
```

```bash
uvicorn myproject.asgi:application --workers 4
# or daphne, hypercorn
```

WSGI (`gunicorn sync/gthread`) will not make async views concurrent on the loop the way ASGI does. Running async views under WSGI falls back to wrappers and loses the point.

## Async views

```python
from django.http import JsonResponse
from django.views import View

async def order_list(request):
    qs = Order.objects.filter(active=True).values("id", "total")[:100]
    data = [row async for row in qs]
    return JsonResponse({"orders": data})

class OrderDetail(View):
    async def get(self, request, pk):
        try:
            order = await Order.objects.select_related("customer").aget(pk=pk)
        except Order.DoesNotExist:
            return JsonResponse({"error": "not found"}, status=404)
        return JsonResponse({
            "id": str(order.id),
            "customer": order.customer_id,  # use id or selected fields
            "total": str(order.total),
        })
```

### ORM async API (illustrative)

| Sync | Async |
|---|---|
| `Model.objects.get` | `aget` |
| `filter().first` | `afirst` |
| `exists` | `aexists` |
| `count` | `acount` |
| `save` | `asave` |
| `delete` | `adelete` |
| iteration | `async for` |

Always pair with `select_related` / `prefetch_related` to avoid implicit sync lazy loads → `SynchronousOnlyOperation`.

```python
# BAD
order = await Order.objects.aget(pk=pk)
print(order.customer.name)  # lazy sync fetch

# GOOD
order = await Order.objects.select_related("customer").aget(pk=pk)
print(order.customer.name)
```

## `sync_to_async` for remaining sync code

```python
from asgiref.sync import sync_to_async

@sync_to_async(thread_sensitive=True)
def charge_card(order_id: str) -> str:
    return SyncPaymentSDK().charge(order_id)

async def checkout(request):
    payment_id = await charge_card(order_id)
    ...
```

Batch work into **few** thread hops; do not wrap every micro-call.

## DRF reality check

Many DRF components (default `APIView` dispatch, serializers with nested relations, permission hooks, pagination) are **sync-centric**.

**Pragmatic options:**

1. **Sync DRF viewsets** for CRUD (threadpool / WSGI or ASGI sync path) — honest and fine for ORM-heavy APIs.
2. **Thin async Django views** for high-concurrency I/O endpoints (webhooks fan-out, aggregators) outside DRF.
3. **Async DRF** experiments / custom `AsyncAPIView` — only if team accepts ecosystem friction; keep serializers dumb and I/O in async services.

```python
# Hybrid: async use case called from sync DRF via async_to_sync — only if no running loop conflict
from asgiref.sync import async_to_sync

class AggregatePricesView(APIView):
    def get(self, request):
        data = async_to_sync(price_use_case.execute)(sku=request.GET["sku"])
        return Response(data)
```

Prefer **not** nesting `async_to_sync` under an already-async stack.

## Middleware

- Fully async middleware must be careful with `sync_to_async` for legacy session/auth.
- Prefer known-good stacks (Django session/auth versions that document async).
- When in doubt, keep middleware thin and sync; do heavy I/O in views/use cases.

## Signals

Async receivers exist in modern Django but ordering/transaction guarantees differ. Prefer explicit calls from use cases over async signal chains for critical business logic.

## Transactions

```python
from django.db import transaction
from asgiref.sync import sync_to_async

@sync_to_async
def place_order(dto):
    with transaction.atomic():
        order = Order.objects.create(...)
        OrderItem.objects.bulk_create(...)
        return order.id
```

Do not open `atomic()` in sync thread A and continue awaits that touch the same connection on the loop.

## Clean architecture (async-aware)

```
async View (HTTP)
  → async UseCase (TaskGroup, timeouts)
      → async Service methods (async ORM / httpx)
      → adapters: sync_to_async(legacy_sdk)
```

DTOs stay frozen dataclasses; use cases return `Output(success=..., error=...)` — same as sync clean architecture. Only the I/O surface becomes async.

```python
@dataclass(frozen=True, slots=True)
class CreateOrderOutput:
    success: bool
    order_id: UUID | None = None
    error: str | None = None

class CreateOrderUseCase:
    def __init__(self, orders: OrderService, inventory: InventoryService):
        self._orders = orders
        self._inventory = inventory

    async def execute(self, input_dto: CreateOrderInput) -> CreateOrderOutput:
        ok, err = await self._inventory.avalidate(input_dto.items)
        if not ok:
            return CreateOrderOutput(success=False, error=err)
        order = await self._orders.acreate(input_dto)
        return CreateOrderOutput(success=True, order_id=order.id)
```

## When Django should stay sync

- Admin-heavy internal tools
- Complex DRF viewsets + nested writes
- Team not on ASGI
- No concurrent outbound I/O benefit

Async Django pays off for **I/O fan-out and long-lived connections**, not for rewriting every CRUD.

## Django/DRF checklist

- [ ] ASGI deployment if shipping async views
- [ ] No lazy ORM in async paths
- [ ] `select_related` / `prefetch_related` before leave-query
- [ ] Thread-sensitive wraps for remaining sync ORM/SDK
- [ ] DRF choice conscious (sync vs thin async views)
- [ ] Pool sizes × workers < DB limit
- [ ] Tests cover `SynchronousOnlyOperation` absence