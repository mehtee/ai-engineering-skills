# Taskiq + FastAPI

Package: [`taskiq-fastapi`](https://github.com/taskiq-python/taskiq-fastapi).

## Init

Call once next to broker definition:

```python
# app/tkq.py
from taskiq_aio_pika import AioPikaBroker
import taskiq_fastapi

broker = AioPikaBroker("amqp://guest:guest@localhost:5672")
taskiq_fastapi.init(broker, "app.main:app")  # uvicorn-style path or factory
```

`init` wires FastAPI dependency context into Taskiq so worker tasks can resolve the same deps as HTTP handlers (with a **mocked** `Request` / `HTTPConnection` — not the original request that enqueued the task).

## Two rules

1. Any base dep that takes `Request` or `HTTPConnection` must mark them with `TaskiqDepends`.
2. Inside `@broker.task`, use only `TaskiqDepends` (not `fastapi.Depends`).

```python
from typing import Annotated
from fastapi import Request
from taskiq import TaskiqDepends

async def get_redis(request: Annotated[Request, TaskiqDepends()]):
    return request.app.state.redis

# or: request: Request = TaskiqDepends()
```

```python
@broker.task
async def job(redis=TaskiqDepends(get_redis)) -> None:
    await redis.set("k", "v")
```

## Lifespan (startup broker on API only)

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.tkq import broker

@asynccontextmanager
async def lifespan(app: FastAPI):
    if not broker.is_worker_process:
        await broker.startup()
    yield
    if not broker.is_worker_process:
        await broker.shutdown()

app = FastAPI(lifespan=lifespan)
```

Worker process imports the app path for deps; `is_worker_process` prevents double connection pools on the API process when the worker loads the same module graph.

## Enqueue from routes

```python
from fastapi import APIRouter
from app.tasks import send_email

router = APIRouter()

@router.post("/signup")
async def signup(email: str):
    await send_email.kiq(email)
    return {"status": "queued"}
```

Do not block the request on `wait_result` unless the product truly needs sync completion; prefer 202 + polling/webhook.

## Sharing app.state resources

Populate heavy clients in FastAPI lifespan **and** mirror them for workers via Taskiq events if the worker does not load full lifespan:

```python
from taskiq import TaskiqEvents, TaskiqState

@broker.on_event(TaskiqEvents.WORKER_STARTUP)
async def worker_resources(state: TaskiqState) -> None:
    state.http = httpx.AsyncClient()
    # or open DB pools needed only in workers
```

Prefer deps that read from `Context.state` or from the mocked `request.app` after `taskiq_fastapi.init`.

## Testing with FastAPI

```python
import taskiq_fastapi
import pytest
from app.tkq import broker
from app.main import create_app

@pytest.fixture
def fastapi_app():
    return create_app()

@pytest.fixture(autouse=True)
def init_taskiq_deps(fastapi_app):
    taskiq_fastapi.populate_dependency_context(broker, fastapi_app)
    yield
    broker.custom_dependency_context = {}
```

Required when tests use `InMemoryBroker` (not a real worker process). Also switch broker via `ENVIRONMENT=pytest` (see testing.md).

## Project sketch

```
app/
  main.py       # FastAPI app + lifespan
  tkq.py        # broker + taskiq_fastapi.init
  deps.py       # shared deps with TaskiqDepends on Request
  tasks/
    email.py
  api/
    routes.py   # .kiq only
```

## Checklist

- [ ] `taskiq-fastapi` installed and `init(broker, "module:app")` called
- [ ] Request-based deps use `TaskiqDepends`
- [ ] Lifespan guards with `not broker.is_worker_process`
- [ ] Worker: `taskiq worker app.tkq:broker --fs-discover`
- [ ] Tests: `populate_dependency_context` + InMemoryBroker
