---
title: "추천 — Flask"
kind: knowledge
status: current
created_at: 2026-05-26T18:00:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-flask, api-design, recipe, recommendation]
---

# 추천 — Flask

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/recommendation|↗ Java 원본]]**

> 백엔드 = 이벤트 적재 (Celery → Redis ZSET hydrate). 모델은 ML 팀.

---

## 1-3. 흐름 / 이벤트 종류 / DB

(Java 원본 [[../../spring/java-spring/api-design/recommendation#1|§1, §2, §3]] 와 동일)

---

## 2. 이벤트 적재 — Celery 또는 직접 Kafka

```python
# app/services/event_service.py
import uuid
import json
from datetime import datetime, timezone
from typing import Optional
from confluent_kafka import Producer

producer = Producer({"bootstrap.servers": "localhost:9092"})


def record_interaction(
    user_id: Optional[str],
    type_: str,
    target_type: str,
    target_id: str,
    context: dict,
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
        producer.produce(
            "interaction-events",
            key=(user_id or target_id).encode(),         # partition key
            value=json.dumps(event).encode(),
        )
        producer.poll(0)
    except Exception as e:
        log.warning("interaction event publish failed: %s", e)
        # best-effort — 비즈니스 흐름 막지 않음
```

> **Python 한정**: `confluent_kafka.Producer.produce()` 가 async — `poll(0)` 으로 background callback 실행. shutdown 시 `producer.flush()` 필수.

---

## 3. Controller

```python
@event_bp.post("/events")
def record_event():
    payload = RecordEventSchema().load(request.get_json())
    user_id = get_jwt_identity_optional()
    record_interaction(
        user_id, payload["type"],
        payload["targetType"], payload["targetId"],
        payload.get("context", {}),
    )
    return ok("이벤트 적재"), 202
```

> 202 Accepted — best-effort.

---

## 4. 추천 결과 조회 — Redis ZSET hydrate

```python
def for_user(user_id: str, kind: str, limit: int) -> list[dict]:
    key = f"rec:user:{user_id}:{kind.lower()}"
    raw = redis_client.zrevrange(key, 0, limit - 1, withscores=True)

    if not raw:
        # cold start fallback
        return trending_fallback(user_id, kind, limit)

    ids = [pid for pid, _ in raw]
    products = product_repo.find_by_ids(ids)
    by_id = {p.id: p for p in products}

    return [
        {"product": by_id[pid].to_dict(), "score": score}
        for pid, score in raw
        if pid in by_id        # 삭제된 상품 필터
    ]


def trending_fallback(user_id: str, kind: str, limit: int) -> list[dict]:
    if kind == "HOME":
        recent_categories = user_view_repo.recent_categories(user_id, 5)
        if recent_categories:
            return product_stats_repo.top_by_categories(recent_categories, limit)
        return product_stats_repo.top_products_30d(limit)
    return []
```

---

## 5. Spring Batch 동등 — Celery beat

```python
# app/workers/recompute_rec.py
@shared_task(bind=True)
def recompute_user_recommendations(self) -> None:
    active_user_ids = user_view_repo.active_users_in_last_days(7, 10_000)
    for uid in active_user_ids:
        top_items = _compute_for(uid)
        key = f"rec:user:{uid}:home"
        with redis_client.pipeline() as pipe:
            pipe.delete(key)
            for item in top_items:
                pipe.zadd(key, {item["id"]: item["score"]})
            pipe.expire(key, 25 * 3600)
            pipe.execute()


# celery_app.conf.beat_schedule
app.conf.beat_schedule = {
    "recompute-rec": {
        "task": "app.workers.recompute_rec.recompute_user_recommendations",
        "schedule": crontab(hour=4, minute=0),     # 매일 새벽 4시
    },
}
```

> **Python 한정**: `redis_client.pipeline()` 으로 multi-command batch — `transaction=True` 시 MULTI/EXEC.

---

## 6. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/recommendation#8 함정 모음|§8]] 의 10 함정 그대로)

### Python 한정

### 함정 11 — `producer.flush()` 누락
async produce → process 종료 시 미전송. `atexit` 또는 lifecycle hook.

### 함정 12 — `redis.zrevrange(..., withscores=True)` return tuple
`[(name, score), ...]` — bytes 일지 str 일지 `decode_responses` 설정에 따름.

---

## 7. 운영 체크리스트

(Java 원본 §9 그대로)

---

## 8. 관련

- [[../../spring/java-spring/api-design/recommendation|↗ Java 원본]]
- [[feed-timeline]]
- [[api-design|↑ hub]]
