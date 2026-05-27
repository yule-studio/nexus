---
title: "추천 — FastAPI"
kind: knowledge
status: current
created_at: 2026-05-27T00:30:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, api-design, recipe, recommendation]
---

# 추천 — FastAPI

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/recommendation|↗ Java 원본]]** · **[[../../python-flask/api-design/recommendation|↗ Flask 변형]]**

> async event 적재 (aiokafka 또는 redis stream) + Celery beat + redis ZSET hydrate.

---

## 1-3. 흐름 / 이벤트 / DB

(Java 원본 [[../../spring/java-spring/api-design/recommendation#1|§1, §2, §3]] 와 동일)

---

## 2. 이벤트 적재 — aiokafka

```bash
pip install aiokafka
```

```python
# app/services/event_service.py
import uuid
import json
from datetime import datetime, timezone
from aiokafka import AIOKafkaProducer

from app.config import settings


_producer: AIOKafkaProducer | None = None


async def start_producer():
    global _producer
    _producer = AIOKafkaProducer(bootstrap_servers=settings.KAFKA_BOOTSTRAP)
    await _producer.start()


async def stop_producer():
    if _producer:
        await _producer.stop()


async def record_interaction(
    user_id: int | None, type_: str, target_type: str, target_id: str, context: dict,
) -> None:
    event = {
        "id": str(uuid.uuid4()),
        "user_id": user_id,
        "type": type_,
        "target_type": target_type,
        "target_id": target_id,
        "context": context,
        "occurred_at": datetime.now(timezone.utc).isoformat(),
    }
    try:
        await _producer.send_and_wait(
            "interaction-events",
            value=json.dumps(event).encode(),
            key=(str(user_id) if user_id else target_id).encode(),
        )
    except Exception as e:
        log.warning("interaction event publish failed: %s", e)


# app/main.py lifespan
@asynccontextmanager
async def lifespan(app: FastAPI):
    await start_producer()
    yield
    await stop_producer()
```

> **`aiokafka` idiom**: `await producer.send_and_wait(...)` — async fire-and-forget 또는 `send` (no wait).

---

## 3. Controller

```python
@router.post("/events", status_code=202)
async def record_event(
    payload: RecordEventRequest,
    user: Annotated[User | None, Depends(get_optional_user)],
):
    await record_interaction(
        user.id if user else None,
        payload.type, payload.targetType, payload.targetId,
        payload.context or {},
    )
    return {"accepted": True}
```

---

## 4. 추천 결과 조회 (redis async)

```python
from redis import asyncio as aioredis


async def for_user(
    redis: aioredis.Redis, db: AsyncSession,
    user_id: int, kind: str, limit: int,
) -> list[dict]:
    key = f"rec:user:{user_id}:{kind.lower()}"
    raw = await redis.zrevrange(key, 0, limit - 1, withscores=True)

    if not raw:
        return await trending_fallback(db, user_id, kind, limit)

    ids = [pid for pid, _ in raw]
    products = await product_repo.find_by_ids(db, ids)
    by_id = {p.id: p for p in products}

    return [
        {"product": by_id[pid].model_dump(), "score": score}
        for pid, score in raw
        if pid in by_id
    ]


async def trending_fallback(db, user_id: int, kind: str, limit: int) -> list[dict]:
    if kind == "HOME":
        recent_cats = await user_view_repo.recent_categories(db, user_id, 5)
        if recent_cats:
            return await product_stats_repo.top_by_categories(db, recent_cats, limit)
        return await product_stats_repo.top_products_30d(db, limit)
    return []
```

---

## 5. Celery beat (sync — DB / redis sync 사용)

(Flask 변형 [[../../python-flask/api-design/recommendation#5 Spring Batch 동등|§5]] 와 동일 — Celery 가 FastAPI 와 무관)

---

## 6. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/recommendation#8 함정 모음|§8]] + Flask 변형 함정)

### FastAPI 한정

### 함정 11 — `aiokafka` producer 가 lifespan 밖 사용
import 시점에 start = race. **lifespan startup 에서 start, shutdown 에서 stop**.

### 함정 12 — `await producer.send_and_wait` vs `await producer.send`
`send_and_wait` = block until ack. `send` = fire-and-forget (더 빠름 but loss 가능). **best-effort 면 `send` + try/except**.

---

## 7. 운영 체크리스트

(Java 원본 §9 + Python async)

---

## 8. 관련

- [[../../spring/java-spring/api-design/recommendation|↗ Java 원본]]
- [[../../python-flask/api-design/recommendation|↗ Flask 변형]]
- [[feed-timeline]]
- [[api-design|↑ hub]]
