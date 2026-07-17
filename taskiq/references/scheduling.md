# Scheduling tasks

Scheduler **only enqueues** messages via the broker. Workers execute them. Run **one** scheduler process in production.

## Label schedules (cron on the task)

```python
from taskiq import TaskiqScheduler
from taskiq.schedule_sources import LabelScheduleSource
from taskiq_aio_pika import AioPikaBroker

broker = AioPikaBroker("amqp://guest:guest@localhost:5672/")

scheduler = TaskiqScheduler(
    broker=broker,
    sources=[LabelScheduleSource(broker)],
)

@broker.task(schedule=[{"cron": "*/5 * * * *", "args": [1]}])
async def heavy_task(value: int) -> int:
    return value + 1
```

```bash
taskiq scheduler app.tkq:scheduler
taskiq scheduler app.tkq:scheduler --skip-first-run  # wait until next minute
```

Cron without timezone is treated as **UTC**. Use `cron_offset` (`"Europe/Berlin"` or `timedelta`) on schedule entries / `ScheduledTask` when the source supports it.

## Dynamic schedules (Redis list source)

```python
from taskiq import TaskiqScheduler
from taskiq.schedule_sources import LabelScheduleSource
from taskiq_redis import ListRedisScheduleSource  # package taskiq-redis

redis_source = ListRedisScheduleSource("redis://localhost")

scheduler = TaskiqScheduler(
    broker=broker,
    sources=[
        LabelScheduleSource(broker),  # keep if you still use decorator schedules
        redis_source,
    ],
)
```

```python
import datetime

await redis_source.startup()

await my_task.schedule_by_time(
    redis_source,
    datetime.datetime.now(datetime.UTC) + datetime.timedelta(minutes=1),
    arg1,
    kw=2,
)

# cron-style dynamic registration also available on sources that support it
```

Always include `LabelScheduleSource` if any tasks define `schedule=` in labels.

## Multiple sources

Pass several sources into `TaskiqScheduler`. Merge functions (`taskiq.scheduler.merge_functions`):

- `preserve_all` — concatenate
- `only_unique` — skip duplicates

Custom merge: pass `merge_func=` to the scheduler for filtering/conflict policy.

## Interval / “every N minutes”

Prefer cron (`*/5 * * * *`) on labels, or dynamic `schedule_by_time` loops that re-register the next run from the task body (careful with failures). There is no separate Celery-beat “interval” type beyond cron/time fields on `ScheduledTask`.

## Pitfalls

| Mistake | Result |
|---|---|
| N scheduler replicas | Task runs ~N times |
| Only Redis source, cron on `@broker.task(schedule=...)` | Labels never read |
| Expecting scheduler to run code | It only `kiq`s |
| Naive local datetimes | Unexpected fire times — use UTC or tz-aware |
| First-minute double fire on deploy | Use `--skip-first-run` |

## Ops

```
Process A: web (startup broker client)
Process B: taskiq worker ...
Process C: taskiq scheduler ...   # exactly one
Infra: RabbitMQ/NATS/… + Redis (results and/or schedule source)
```
