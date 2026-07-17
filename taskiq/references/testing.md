# Testing Taskiq

## Swap broker under pytest

```python
# app/tkq.py
import os
from taskiq import InMemoryBroker, AsyncBroker

broker: AsyncBroker
if os.environ.get("ENVIRONMENT") == "pytest":
    broker = InMemoryBroker()
else:
    broker = make_prod_broker()
```

```toml
# pyproject.toml
[tool.pytest.ini_options]
env = ["ENVIRONMENT=pytest"]
```

Or export `ENVIRONMENT=pytest` in CI. Use `pytest-env` if needed.

## Async tests

```python
# conftest.py
import pytest

@pytest.fixture
def anyio_backend():
    return "asyncio"
```

```python
import pytest
from app.tasks import parse_int

@pytest.mark.anyio
async def test_task_body():
    assert await parse_int("11") == 11  # call like a normal function
```

## Code that uses `.kiq()`

With `InMemoryBroker`, tasks run in-process and results work:

```python
@pytest.mark.anyio
async def test_enqueue_path():
    from app.tasks import parse_int
    handle = await parse_int.kiq("21")
    result = await handle.wait_result()
    assert result.return_value == 21
```

## dependency_overrides

Brokers support overriding deps for tests (same idea as FastAPI):

```python
# pattern — override a TaskiqDepends provider
async def fake_redis():
    return FakeRedis()

broker.dependency_overrides[get_redis] = fake_redis
# run tests...
broker.dependency_overrides.clear()
```

Confirm exact API against installed `taskiq` version if overrides land on broker vs receiver; docs cover `dependency_overrides` for testing DI graphs.

## FastAPI + InMemoryBroker

Worker never runs, so FastAPI context is missing unless populated:

```python
import taskiq_fastapi

@pytest.fixture(autouse=True)
def init_taskiq_deps(fastapi_app):
    taskiq_fastapi.populate_dependency_context(broker, fastapi_app)
    yield
    broker.custom_dependency_context = {}
```

## Django

- Use `ENVIRONMENT=pytest` → InMemoryBroker.
- Mark DB tests with `pytest.mark.django_db`.
- Prefer testing task **bodies** with ORM fixtures; integration-test `.kiq()` only when needed.

## What not to do

- Hit real RabbitMQ/Redis in unit tests by default.
- Start `taskiq worker` subprocess for pure unit tests.
- Assert on DummyResultBackend return values.
