---
title: "분산 락 (redis.asyncio.lock) — FastAPI"
kind: knowledge
status: current
created_at: 2026-05-26T23:40:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, api-design, recipe, distributed-lock]
---

# 분산 락 (redis.asyncio.lock) — FastAPI

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/distributed-lock|↗ Java 원본]]** · **[[../../python-flask/api-design/distributed-lock|↗ Flask 변형]]**

---

## 1-2-6-7. JVM vs 분산 / 시나리오 / fencing / TTL 함정

(Java 원본 [[../../spring/java-spring/api-design/distributed-lock#1|§1, §2, §6, §7]] 와 동일)

---

## 3-4. 의존성 + async lock

```python
from redis import asyncio as aioredis
from contextlib import asynccontextmanager


@asynccontextmanager
async def distributed_lock(
    redis: aioredis.Redis, key: str,
    timeout: int = 5, blocking_timeout: int = 3,
):
    """async context manager — Java try-with-resources 동등."""
    lock = redis.lock(key, timeout=timeout, blocking_timeout=blocking_timeout)
    acquired = False
    try:
        acquired = await lock.acquire()
        if not acquired:
            raise BusinessException(
                429, "RATE_LIMIT_EXCEEDED", f"락 획득 실패: {key}",
            )
        yield lock
    finally:
        if acquired:
            try:
                await lock.release()
            except aioredis.exceptions.LockNotOwnedError:
                # TTL 만료 후 다른 worker — 무시
                pass


# 사용
async def decrease_stock(
    db: AsyncSession, redis: aioredis.Redis,
    option_id: str, qty: int,
) -> None:
    async with distributed_lock(redis, f"lock:option:{option_id}", timeout=5):
        async with db.begin():
            option = await db.get(ProductOption, option_id)
            if option is None:
                raise NotFoundException("option")
            option.stock -= qty
```

> **FastAPI idiom**: `async with` 중첩 — lock 안 transaction.

---

## 5. Dependency 식 (Depends + factory)

```python
def with_lock(key_fn: Callable[..., str], timeout: int = 5):
    """async dependency factory."""
    async def _dep(redis: Annotated[aioredis.Redis, Depends(get_redis)], **kwargs):
        key = key_fn(**kwargs)
        async with distributed_lock(redis, f"lock:{key}", timeout=timeout) as lock:
            yield lock
    return _dep


# 사용 — generator dependency
@router.post("/options/{option_id}/decrease")
async def decrease_stock_endpoint(
    option_id: str,
    qty: int,
    db: SessionDep,
    _lock: Annotated[Any, Depends(with_lock(lambda option_id, **_: f"option:{option_id}"))],
):
    await stock_service.decrease(db, option_id, qty)
    return {"ok": True}
```

> **Python 한정**: Depends 안에서 `async with` 를 yield 하려면 generator dependency 패턴. 함수가 `async def` + `yield`.

---

## 8. @scheduled 분산 — Celery beat + redbeat

(Flask 변형 [[../../python-flask/api-design/distributed-lock#8|§8]] 와 동일 — Celery beat 가 FastAPI 와 무관하게 사용)

---

## 9. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/distributed-lock#10 함정 모음|§10]] 의 10 함정 그대로)

### Python/FastAPI 한정

### 함정 11 — sync `redis.lock` + async endpoint
sync `redis.Redis.lock` 을 async endpoint 안에서 = event loop block. **`redis.asyncio` lock** 만.

### 함정 12 — Depends generator + raise
`yield` 후 raise 면 cleanup 안 실행. **try/except/finally 안에서 yield**.

### 함정 13 — `async with lock` 내부 long-running CPU
lock TTL 보다 길어지면 다른 worker 가 잡음. CPU heavy = `asyncio.to_thread` 로 thread pool 분리.

---

## 10. 운영 체크리스트

(Java 원본 §11 + Python)

- [ ] `redis.asyncio` lock 만 (sync 혼용 X)
- [ ] `async with distributed_lock(...)` context manager
- [ ] Depends generator 의 try/finally 정확히
- [ ] Celery beat 는 redbeat 등 분산 backend

---

## 11. 관련

- [[../../spring/java-spring/api-design/distributed-lock|↗ Java 원본]]
- [[../../python-flask/api-design/distributed-lock|↗ Flask 변형]]
- [[cache-redis]]
- [[api-design|↑ hub]]
