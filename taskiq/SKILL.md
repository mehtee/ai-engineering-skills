---
name: taskiq
description: >
  Distributed async task queue for Python (Taskiq). Use whenever the user mentions
  Taskiq, background/async tasks, workers, .kiq(), task brokers (RabbitMQ/Redis/NATS/ZMQ),
  result backends, TaskiqScheduler, cron/interval schedules, taskiq-fastapi, Celery-like
  work in async FastAPI/Django/DRF apps, or migrating from Celery/arq/Dramatiq to an
  asyncio-native queue. Prefer this skill over generic Celery advice for async Python.
---

# Taskiq

Async-first distributed task library. Core is modular: **broker** (kick/listen) + optional
**result backend** + **scheduler** + **middlewares** + FastAPI-style **TaskiqDepends**.

Official docs live in this repo under `docs/`. Ecosystem packages: `taskiq-*` on PyPI
(e.g. `taskiq-aio-pika`, `taskiq-redis`, `taskiq-nats`, `taskiq-fastapi`).

## When to choose Taskiq

| Prefer Taskiq | Prefer something else |
|---|---|
| Async def tasks, FastAPI, modern asyncio stack | Fully sync monolith → Celery/Dramatiq |
| Pluggable brokers (AMQP, Redis, NATS, SQS, …) | Need built-in task abort only → check arq/streaQ |
| DI + schedules + middlewares without Celery weight | |

## Progressive loading

Read only what the task needs:

| File | When |
|---|---|
| [references/fastapi.md](references/fastapi.md) | FastAPI / Starlette apps, lifespan, shared deps |
| [references/django-drf.md](references/django-drf.md) | Django or DRF (ORM, settings, ASGI/WSGI) |
| [references/scheduling.md](references/scheduling.md) | cron, interval, dynamic schedules, redis source |
| [references/testing.md](references/testing.md) | pytest, InMemoryBroker, dependency_overrides |
| [references/ecosystem.md](references/ecosystem.md) | Broker/result packages, middlewares, CLI flags |

## Mental model

```
Client:  task.kiq(*args) → Kicker builds message → Broker.kick → queue
Worker:  Broker.listen → execute → ResultBackend.store
Client:  task.wait_result() / is_ready()  (needs real result backend)
Scheduler process: sources → broker.kick only (does NOT execute)
```

Always call `await broker.startup()` on the **client** before `kiq` (and `shutdown` on exit).
Workers call startup via CLI. Skip client startup only when `broker.is_worker_process`.

## Minimal setup

```bash
pip install taskiq
# production-ish defaults:
pip install taskiq-aio-pika taskiq-redis
# FastAPI:
pip install taskiq-fastapi
```

```python
# app/tkq.py  — single broker module (import this everywhere)
import os
from taskiq import InMemoryBroker
from taskiq_aio_pika import AioPikaBroker
from taskiq_redis import RedisAsyncResultBackend

env = os.environ.get("ENVIRONMENT")

if env == "pytest":
    broker = InMemoryBroker()
else:
    broker = AioPikaBroker(
        os.environ.get("AMQP_URL", "amqp://guest:guest@localhost:5672")
    ).with_result_backend(
        RedisAsyncResultBackend(os.environ.get("REDIS_URL", "redis://localhost"))
    )
```

```python
# app/tasks.py
from app.tkq import broker

@broker.task
async def add(a: int, b: int) -> int:
    return a + b

# optional: stable name + labels
@broker.task(task_name="reports.generate", timeout=30)
async def generate_report(user_id: int) -> str:
    ...
```

```python
# enqueue from app code
from app.tasks import add

async def somewhere():
    task = await add.kiq(1, 2)
    result = await task.wait_result()
    if result.is_err:
        raise result.error
    return result.return_value
```

```bash
# discover tasks: either pass modules or use -fsd
taskiq worker app.tkq:broker app.tasks
taskiq worker app.tkq:broker --fs-discover --tasks-pattern '**/tasks.py'
```

## Task API cheatsheet

```python
# fire-and-forget / await result
handle = await my_task.kiq(arg, kw=1)
ready = await handle.is_ready()
result = await handle.wait_result(timeout=10)  # TaskiqResult

# kicker: labels, broker override, custom id
await my_task.kicker().with_labels(delay=5, timeout=0.5).with_task_id("id-1").kiq()

# labels on decorator (broker-specific meaning, e.g. delay for some brokers)
@broker.task(timeout=5, schedule=[{"cron": "0 * * * *"}])
async def hourly(): ...

# shared tasks (library code) — bind broker later
from taskiq import async_shared_broker
@async_shared_broker.task
async def shared(): ...
async_shared_broker.default_broker(broker)
```

`TaskiqResult` fields: `return_value`, `is_err`, `error`, `execution_time`, `labels`.

## State, events, dependencies

```python
from taskiq import Context, TaskiqDepends, TaskiqEvents, TaskiqState

@broker.on_event(TaskiqEvents.WORKER_STARTUP)
async def on_worker_start(state: TaskiqState) -> None:
    state.db_pool = await create_pool()

@broker.on_event(TaskiqEvents.WORKER_SHUTDOWN)
async def on_worker_stop(state: TaskiqState) -> None:
    await state.db_pool.close()

def get_pool(ctx: Context = TaskiqDepends()):
    return ctx.state.db_pool

@broker.task
async def use_db(pool=TaskiqDepends(get_pool)) -> None:
    ...
```

Events: `WORKER_STARTUP`, `WORKER_SHUTDOWN`, `CLIENT_STARTUP`, `CLIENT_SHUTDOWN`.

Rules:

- Prefer `TaskiqDepends` over raw `Context` for typed injection.
- Generators/async generators work as deps (setup/teardown).
- Use `broker.add_event_handler(event, fn)` when registering dynamically.

## Project layout (recommended)

```
project/
  app/
    tkq.py          # broker + scheduler + taskiq_fastapi.init
    tasks/          # @broker.task modules (auto-discover **/tasks.py)
    api/            # FastAPI/DRF — only .kiq(), never heavy work
  manage.py / main.py
```

Keep broker construction in **one module** so worker, web, and scheduler import the same object.

## Framework quick pointers

**FastAPI** — `taskiq_fastapi.init(broker, "pkg.main:app")`; mark `Request`/`HTTPConnection` with `TaskiqDepends`; lifespan starts broker only when `not broker.is_worker_process`. Details: [references/fastapi.md](references/fastapi.md).

**Django / DRF** — no official `taskiq-django` package; wire broker in AppConfig / ASGI lifespan, use sync ORM via `sync_to_async` or thread deps, configure `DJANGO_SETTINGS_MODULE` for workers. Details: [references/django-drf.md](references/django-drf.md).

## CLI essentials

```bash
taskiq worker path.to.module:broker_var [task_modules...]
  --fs-discover / -fsd
  --tasks-pattern / -tp '**/tasks.py'   # repeatable
  --ack-type when_received|when_executed|when_saved|manual
  --workers / -w N
  --use-process-pool                   # CPU-bound sync tasks
  --max-threadpool-threads N
  --no-parse                           # disable param type casts
  --reload                             # dev hot reload (extra)

taskiq scheduler path.to.module:scheduler [--skip-first-run]
# Run exactly ONE scheduler process in production.
```

Per-task ack override: `@broker.task(ack_type="when_saved")`. Manual: inject `Context` and `await context.ack()`.

## Middlewares (built-in)

```python
from taskiq.middlewares import SimpleRetryMiddleware, SmartRetryMiddleware, PrometheusMiddleware

broker.add_middleware(SimpleRetryMiddleware(default_retry_count=3))
# or SmartRetryMiddleware for exponential backoff / richer control
```

Custom middleware: subclass `TaskiqMiddleware` (see `docs/extending-taskiq/middleware.md`).

## Common pitfalls

1. **No `startup()`** on client → undefined broker behavior.
2. **`InMemoryBroker` in multi-process** → not distributed; tests/local only.
3. **`ZeroMQBroker` with `-w > 1`** → each worker executes every message; use `-w 1`.
4. **Multiple schedulers** → duplicate fires; run one.
5. **Cron in labels without `LabelScheduleSource`** → schedules ignored.
6. **Waiting on results with DummyResultBackend** → empty/None results.
7. **FastAPI deps without `TaskiqDepends` on Request** → resolution fails in worker.
8. **Import side effects** → defining broker differently in worker vs web splits registries.
9. **Sync CPU work on default threadpool** → use `--use-process-pool` or keep tasks async/IO.
10. **`is_worker_process` forgotten in web lifespan** → worker double-inits app resources.

## Implementation checklist

When adding Taskiq to a project:

1. Add deps: `taskiq` + broker package + result backend (+ `taskiq-fastapi` if FastAPI).
2. Create `tkq.py` with env-switched broker (`InMemoryBroker` for pytest).
3. Define tasks under discoverable modules; use explicit `task_name` for stable routing.
4. Wire web lifespan/AppConfig startup/shutdown with `is_worker_process` guard.
5. Document process model: API, `taskiq worker`, optional `taskiq scheduler`.
6. Add tests per [references/testing.md](references/testing.md).
7. Production: real broker + result backend, one scheduler, ack_type policy, retries.

## Docs map (this repository)

- Guide: `docs/guide/` (getting-started, architecture, CLI, state-and-deps, scheduling, testing, dynamic-brokers)
- Components: `docs/available-components/`
- Integrations: `docs/framework_integrations/`
- Extending: `docs/extending-taskiq/`
- Examples: `docs/examples/`
