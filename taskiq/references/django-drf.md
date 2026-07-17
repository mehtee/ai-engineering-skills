# Taskiq + Django / DRF

There is **no official `taskiq-django` package** in Taskiq docs. Integrate manually: one broker module, Django settings for URLs, worker process with `DJANGO_SETTINGS_MODULE`, careful ORM usage.

Use this reference when the web stack is Django or Django REST Framework and background work should be async-native (email, webhooks, report generation, fan-out IO).

## Settings

```python
# myproject/settings.py
import os

TASKIQ_AMQP_URL = os.environ.get("AMQP_URL", "amqp://guest:guest@localhost:5672")
TASKIQ_REDIS_URL = os.environ.get("REDIS_URL", "redis://localhost/0")
# optional feature flag for tests
TASKIQ_INMEMORY = os.environ.get("ENVIRONMENT") == "pytest"
```

## Broker module

```python
# myproject/tkq.py
import django
from django.conf import settings
from taskiq import InMemoryBroker

# Ensure Django is configured when worker imports this module first
if not settings.configured:
    import os
    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "myproject.settings")
    django.setup()

if getattr(settings, "TASKIQ_INMEMORY", False):
    broker = InMemoryBroker()
else:
    from taskiq_aio_pika import AioPikaBroker
    from taskiq_redis import RedisAsyncResultBackend

    broker = AioPikaBroker(settings.TASKIQ_AMQP_URL).with_result_backend(
        RedisAsyncResultBackend(settings.TASKIQ_REDIS_URL)
    )
```

Import `myproject.tkq` only **after** settings are available, or call `django.setup()` as above so worker CLI works:

```bash
export DJANGO_SETTINGS_MODULE=myproject.settings
taskiq worker myproject.tkq:broker myapp.tasks --fs-discover
```

## AppConfig startup (WSGI / runserver)

```python
# myapp/apps.py
from django.apps import AppConfig

class MyAppConfig(AppConfig):
    name = "myapp"

    def ready(self) -> None:
        # Import tasks so @broker.task registers on worker & scheduler
        from myapp import tasks  # noqa: F401

        # Optional: client startup is better in ASGI lifespan;
        # for pure WSGI, start broker lazily before first .kiq()
```

Prefer **ASGI lifespan** when using Daphne/Uvicorn/Hypercorn:

```python
# myproject/asgi.py
import os
from contextlib import asynccontextmanager
from django.core.asgi import get_asgi_application

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "myproject.settings")
django_app = get_asgi_application()

from myproject.tkq import broker  # after setup

@asynccontextmanager
async def lifespan(app):
    if not broker.is_worker_process:
        await broker.startup()
    try:
        yield
    finally:
        if not broker.is_worker_process:
            await broker.shutdown()

# If using a pure ASGI stack wrapper, attach lifespan there.
# With channels / starlette mount patterns, wrap accordingly.
```

For classic WSGI + sync views, call startup once (module-level asyncio runner is fragile). Pattern:

```python
# myproject/tkq_client.py
import asyncio
from myproject.tkq import broker

_started = False

async def _ensure():
    global _started
    if not _started and not broker.is_worker_process:
        await broker.startup()
        _started = True

def kiq(task, *args, **kwargs):
    async def _run():
        await _ensure()
        return await task.kiq(*args, **kwargs)
    return asyncio.run(_run())  # only safe outside running loop
```

Inside **async views** / async DRF: `await broker.startup()` once in lifespan, then `await task.kiq(...)`.

## Tasks and the ORM

Django ORM is sync. In async tasks:

```python
from asgiref.sync import sync_to_async
from myproject.tkq import broker
from myapp.models import User

@broker.task
async def deactivate_user(user_id: int) -> None:
    @sync_to_async
    def _work():
        User.objects.filter(pk=user_id).update(is_active=False)
    await _work()
```

Or define a **sync** task and let the worker run it in the thread/process pool:

```python
@broker.task
def deactivate_user_sync(user_id: int) -> None:
    User.objects.filter(pk=user_id).update(is_active=False)
```

Rules:

- Never call ORM across `await` without `sync_to_async` / thread (connection state).
- Open/close DB connections carefully in long workers; Django’s `close_old_connections()` around sync work helps.
- Prefer passing **IDs** into tasks, not model instances (serialization).

## DRF views

```python
# async-capable
from rest_framework.decorators import api_view
from rest_framework.response import Response
from myapp.tasks import deactivate_user

@api_view(["POST"])
async def deactivate(request, pk: int):
    await deactivate_user.kiq(pk)
    return Response({"queued": True}, status=202)
```

Sync DRF:

```python
@api_view(["POST"])
def deactivate(request, pk: int):
    # bridge to async kiq — prefer helper that uses running loop if present
    import asyncio
    from myproject.tkq import broker

    async def _enqueue():
        if not broker.is_worker_process:
            # assume lifespan already started in ASGI; else await broker.startup()
            await deactivate_user.kiq(pk)
    asyncio.get_event_loop().create_task(_enqueue())  # only if loop running
    return Response({"queued": True}, status=202)
```

Cleaner approach for sync Django: keep a small **async helper module** and use `asgiref.sync.async_to_sync`:

```python
from asgiref.sync import async_to_sync
from myapp.tasks import deactivate_user

async_to_sync(deactivate_user.kiq)(pk)
```

Ensure `broker.startup()` completed before first `kiq` (ASGI lifespan or management command).

## Dependencies with Django settings / clients

```python
from django.conf import settings
from taskiq import TaskiqDepends, Context, TaskiqEvents, TaskiqState
from myproject.tkq import broker

@broker.on_event(TaskiqEvents.WORKER_STARTUP)
async def setup(state: TaskiqState) -> None:
    state.api_key = settings.EXTERNAL_API_KEY

def api_key(ctx: Context = TaskiqDepends()) -> str:
    return ctx.state.api_key

@broker.task
async def call_vendor(key: str = TaskiqDepends(api_key)) -> None:
    ...
```

## Management commands

```python
# myapp/management/commands/run_taskiq_worker_hint.py
from django.core.management.base import BaseCommand

class Command(BaseCommand):
    help = "Print recommended taskiq worker command"

    def handle(self, *args, **options):
        self.stdout.write(
            "DJANGO_SETTINGS_MODULE=myproject.settings "
            "taskiq worker myproject.tkq:broker myapp.tasks -fsd"
        )
```

Scheduler (one process):

```bash
taskiq scheduler myproject.tkq:scheduler
```

## Testing

```python
# conftest.py / settings override
# ENVIRONMENT=pytest → InMemoryBroker in tkq.py

import pytest
from myapp.tasks import deactivate_user

@pytest.mark.django_db
@pytest.mark.anyio
async def test_deactivate(user):
    # call underlying function without broker
    await deactivate_user(user.id)
```

For code paths that `.kiq()`, use InMemoryBroker so `wait_result` works in-process.

## Checklist

- [ ] `DJANGO_SETTINGS_MODULE` set for worker/scheduler processes
- [ ] Tasks imported on worker start (AppConfig.ready or CLI module list / fs-discover)
- [ ] ORM via sync tasks or `sync_to_async`; pass PKs not instances
- [ ] `broker.startup()` on web process before `kiq`
- [ ] `is_worker_process` guard so workers do not open HTTP-only resources twice
- [ ] Single scheduler process if using cron labels / redis schedules
