# Ecosystem & components

## Brokers (selection)

| Broker | Package | Notes |
|---|---|---|
| InMemoryBroker | `taskiq` | Local/tests; not multi-process |
| ZeroMQBroker | `taskiq` / `taskiq[zmq]` | Single worker only (`-w 1`) |
| AioPika (RabbitMQ) | `taskiq-aio-pika` | Production default candidate |
| Redis | `taskiq-redis` | Broker + result + schedule sources |
| NATS | `taskiq-nats` | Broker + result backends |
| SQS | `taskiq-aio-sqs` (third-party) | AWS |

Core abstract API: `AsyncBroker.kick` / `listen`. Attach results: `broker.with_result_backend(backend)`.

## Result backends

| Backend | Package |
|---|---|
| DummyResultBackend | built-in (default; no real storage) |
| InMemoryResultBackend | with InMemoryBroker |
| Redis | `taskiq-redis` (`RedisAsyncResultBackend`) |
| NATS | `taskiq-nats` |
| PostgreSQL | `taskiq-postgresql` (third-party) |

Without a real backend, `wait_result` / timings are not trustworthy.

## Schedule sources

| Source | Role |
|---|---|
| `LabelScheduleSource` | Reads `schedule=` from task labels |
| `ListRedisScheduleSource` | Dynamic add/remove in Redis (`taskiq-redis`) |

See `docs/available-components/schedule-sources.md` for the full list.

## Middlewares

Built-in (`taskiq.middlewares`):

- `SimpleRetryMiddleware` — fixed retry count
- `SmartRetryMiddleware` — smarter backoff / control
- `PrometheusMiddleware` — metrics

Hooks on `TaskiqMiddleware`: client/worker message lifecycle (see extending docs).

```python
broker.add_middleware(SimpleRetryMiddleware(default_retry_count=3))
```

## Acknowledgements

CLI: `--ack-type when_received | when_executed | when_saved | manual`  
Per task: `@broker.task(ack_type="when_saved")`  
Manual: `await context.ack()` with `Context = TaskiqDepends()`  
Requires brokers that yield `AckableMessage`.

## CLI worker options (high signal)

```
taskiq worker module:broker [modules...]
  -fsd / --fs-discover
  -tp / --tasks-pattern   (repeatable; default **/tasks.py)
  -w / --workers
  --ack-type
  --use-process-pool / --max-process-pool-processes
  --max-threadpool-threads
  --no-parse              # disable signature type casting
  --reload                # dev
```

Type casting: `kiq("2")` into `val: int` becomes `2` when parse enabled.

## Serializers / formatters

Pluggable via broker configuration (`TaskiqFormatter`, serializers in `taskiq.serializers`). Use when payloads need orjson/msgpack or custom security.

## Pipelines / gather

```python
from taskiq import gather

# fire multiple tasks; wait as a group (see taskiq.funcs.gather)
results = await gather(
    task_a.kiq(...),
    task_b.kiq(...),
)
```

For ordered multi-step workflows, chain by awaiting `wait_result` then `kiq` next, or use project-level orchestration; check current docs for pipeline helpers as APIs evolve.

## Extending

Implement ABCs from `taskiq.abc`:

- `AsyncBroker`
- `AsyncResultBackend`
- `TaskiqMiddleware`
- `ScheduleSource`

Examples: `docs/examples/extending/`, guides under `docs/extending-taskiq/`.

## Related packages (FastAPI path)

```
pip install taskiq taskiq-aio-pika taskiq-redis taskiq-fastapi
```

## Docs index in this repo

- Brokers: `docs/available-components/brokers.md`
- Results: `docs/available-components/result-backends.md`
- Middlewares: `docs/available-components/middlewares.md`
- Schedules: `docs/available-components/schedule-sources.md`
- CLI: `docs/guide/cli.md`
