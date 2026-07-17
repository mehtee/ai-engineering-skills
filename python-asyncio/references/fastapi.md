# FastAPI + asyncio

FastAPI is async-first on ASGI (Starlette). Correctness still depends on **what your callables await**.

## Route style

```python
from fastapi import FastAPI, Depends, Request, HTTPException
from contextlib import asynccontextmanager
import httpx

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.http = httpx.AsyncClient(
        timeout=httpx.Timeout(5.0, connect=2.0),
        limits=httpx.Limits(max_connections=100, max_keepalive_connections=20),
    )
    yield
    await app.state.http.aclose()

app = FastAPI(lifespan=lifespan)

@app.get("/health")
async def health():
    return {"ok": True}

@app.get("/prices/{sku}")
async def prices(sku: str, request: Request):
    client: httpx.AsyncClient = request.app.state.http
    async with asyncio.TaskGroup() as tg:
        t1 = tg.create_task(client.get(f"https://a/api/{sku}"))
        t2 = tg.create_task(client.get(f"https://b/api/{sku}"))
    # validate responses...
    return {"a": t1.result().json(), "b": t2.result().json()}
```

## `async def` vs `def`

| Definition | Runs on | Use for |
|---|---|---|
| `async def` | Event loop | awaitable I/O, fan-out |
| `def` | Threadpool | Sync ORM/SDK, pure blocking libs |

Mixing: an `async def` route that calls blocking code **blocks the loop**. A `def` route that only does async incorrectly cannot await.

## Dependencies

```python
async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    async with asyncio.timeout(1.0):
        user = await user_repo.aget_by_token(token)
    if not user:
        raise HTTPException(401)
    return user

@app.get("/me")
async def me(user: User = Depends(get_current_user)):
    return user
```

Async deps can depend on sync deps and vice versa; Starlette schedules appropriately. Keep dependency graphs shallow.

## BackgroundTasks — limits

```python
from fastapi import BackgroundTasks

@app.post("/signup")
async def signup(data: Signup, tasks: BackgroundTasks):
    user = await create_user(data)
    tasks.add_task(send_welcome_email, user.id)  # after response, same process
    return {"id": user.id}
```

- Not durable (process crash loses task).
- Shares worker resources with requests.
- Prefer a queue for email/billing/webhooks that must happen.

## Streaming and SSE

```python
from fastapi.responses import StreamingResponse

async def gen():
    async for chunk in upstream.stream():
        yield chunk

@app.get("/stream")
async def stream():
    return StreamingResponse(gen(), media_type="application/json")
```

Keep generators cancellation-safe (`try/finally` close upstream).

## WebSockets

```python
@app.websocket("/ws")
async def ws(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            msg = await websocket.receive_text()
            await websocket.send_text(msg)
    except WebSocketDisconnect:
        await cleanup()
```

## Error handling with TaskGroup

```python
@app.get("/aggregate")
async def aggregate():
    try:
        async with asyncio.TaskGroup() as tg:
            ...
    except* httpx.HTTPError as eg:
        raise HTTPException(502, "upstream failed") from eg
```

## DB patterns (SQLAlchemy 2 async sketch)

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

engine = create_async_engine(url, pool_size=10, max_overflow=20)
Session = async_sessionmaker(engine, expire_on_commit=False)

async def get_session():
    async with Session() as session:
        yield session
```

- One engine per process; never per request.
- `expire_on_commit=False` often reduces lazy-load surprises in async.
- Lazy I/O after session close is a classic bug — load what you need inside the session.

## Production uvicorn

```bash
uvicorn app:app --host 0.0.0.0 --port 8000 --workers 4 --loop uvloop --http httptools
```

- Workers = processes; each has its own loop and pools → size pools **per worker**.
- `uvloop` for performance where supported.
- Set proxy timeouts and app timeouts consistently.
- Graceful shutdown: lifespan closes clients; avoid non-cancellable shields on shutdown path.

## Clean architecture mapping

```
Route (thin) → async UseCase.execute(dto) → async Services / Ports
```

Keep HTTP concerns (status codes, headers) in routes; keep TaskGroup fan-out in use cases; wrap sync SDKs in infrastructure adapters with `to_thread`.

## FastAPI checklist

- [ ] Lifespan-managed clients
- [ ] Timeouts on httpx/DB
- [ ] Bounded concurrency for fan-out
- [ ] Sync work in `def` routes or `to_thread`, not bare in `async def`
- [ ] No critical durability on BackgroundTasks
- [ ] Workers × pool size does not crush DB `max_connections`