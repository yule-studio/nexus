---
title: "Webhook 송신 — FastAPI"
kind: knowledge
status: current
created_at: 2026-05-27T00:40:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, api-design, recipe, webhook]
---

# Webhook 송신 — FastAPI

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/webhook-send|↗ Java 원본]]** · **[[../../python-flask/api-design/webhook-send|↗ Flask 변형]]**

> Outbox (async DB) + Celery worker + httpx async + HMAC-SHA-256.

---

## 1-2-8. 무엇을 / DB / 수신자 가이드

(Java 원본 [[../../spring/java-spring/api-design/webhook-send#1|§1, §2, §8]] 와 동일)

---

## 3. Outbox enqueue (async)

```python
# app/services/webhook_service.py
from sqlalchemy import select
from sqlalchemy.exc import IntegrityError
from sqlalchemy.ext.asyncio import AsyncSession

from app.domain.webhook import WebhookSubscription, WebhookDelivery, DeliveryStatus


async def enqueue_webhook(
    db: AsyncSession, event_type: str, event_id: str, payload: dict,
) -> None:
    subs = await db.scalars(
        select(WebhookSubscription)
        .where(WebhookSubscription.active.is_(True))
        .where(WebhookSubscription.event_types.contains(event_type))
    )
    for sub in subs.all():
        try:
            db.add(WebhookDelivery(
                subscription_id=sub.id,
                event_id=event_id,
                event_type=event_type,
                payload=payload,
                status=DeliveryStatus.PENDING.value,
                attempts=0,
                next_attempt_at=datetime.now(timezone.utc),
            ))
            await db.flush()                   # commit 은 호출자가
        except IntegrityError:
            await db.rollback()                 # (sub_id, event_id) UNIQUE — 멱등
```

---

## 4. Worker — Celery (sync — async 도 가능하지만 Celery 가 sync 친화)

```python
import hashlib
import hmac
import json
import httpx

from celery import shared_task
from sqlalchemy.orm import Session

from app.domain.webhook import WebhookDelivery, WebhookSubscription, DeliveryStatus

_BACKOFF = [30, 300, 1800, 7200, 21600, 43200, 86400]


@shared_task(bind=True)
def process_webhook_deliveries(self) -> None:
    """매 1초 — Celery beat schedule."""
    from app.celery_app import SyncSessionLocal       # Celery 는 sync DB session

    with SyncSessionLocal() as db:
        due = db.query(WebhookDelivery).filter(
            WebhookDelivery.status.in_([DeliveryStatus.PENDING.value, DeliveryStatus.RETRY.value]),
            WebhookDelivery.next_attempt_at <= datetime.now(timezone.utc),
        ).limit(50).all()
        for d in due:
            deliver_one.delay(d.id)


@shared_task(bind=True)
def deliver_one(self, delivery_id: int) -> None:
    with SyncSessionLocal() as db:
        delivery = db.get(WebhookDelivery, delivery_id)
        if delivery is None or delivery.status not in (
            DeliveryStatus.PENDING.value, DeliveryStatus.RETRY.value
        ):
            return

        sub = db.get(WebhookSubscription, delivery.subscription_id)
        if sub is None or not sub.active:
            delivery.status = DeliveryStatus.DEAD_LETTER.value
            db.commit()
            return

        payload_str = json.dumps(delivery.payload, separators=(",", ":"), default=str)
        signature = hmac.new(
            sub.secret.encode(), payload_str.encode(), hashlib.sha256
        ).hexdigest()

        try:
            with httpx.Client(timeout=10.0) as client:
                response = client.post(
                    sub.target_url,
                    content=payload_str,
                    headers={
                        "Content-Type": "application/json",
                        "X-Webhook-Event": delivery.event_type,
                        "X-Webhook-Delivery": str(delivery.id),
                        "X-Webhook-Signature": f"sha256={signature}",
                        "X-Idempotency-Key": delivery.event_id,
                        "User-Agent": "Shop-Webhook/1.0",
                    },
                )
            if 200 <= response.status_code < 300:
                delivery.status = DeliveryStatus.SENT.value
                delivery.sent_at = datetime.now(timezone.utc)
            elif 400 <= response.status_code < 500:
                delivery.status = DeliveryStatus.DEAD_LETTER.value
                delivery.last_response_status = response.status_code
                delivery.last_error = response.text[:500]
            else:
                _schedule_retry(delivery, response.status_code, response.text)
        except Exception as e:
            _schedule_retry(delivery, None, str(e))

        db.commit()


def _schedule_retry(d: WebhookDelivery, status: int | None, error: str | None) -> None:
    d.attempts += 1
    d.last_response_status = status
    d.last_error = (error or "")[:500]
    if d.attempts > len(_BACKOFF):
        d.status = DeliveryStatus.DEAD_LETTER.value
    else:
        d.status = DeliveryStatus.RETRY.value
        d.next_attempt_at = datetime.now(timezone.utc) + timedelta(seconds=_BACKOFF[d.attempts - 1])
```

> **왜 worker 가 sync**: Celery 자체가 sync friendly. async worker (`celery[asyncio]`) 도 있지만 안정성 / 운영성은 sync 가 우위. async FastAPI + sync Celery worker = 표준 조합.

---

## 5. 또는 — FastAPI BackgroundTasks (간단)

작은 webhook 이면 BackgroundTasks 가능 (단 retry / monitoring 없음):

```python
@router.post("/orders")
async def create_order(payload: OrderRequest, bg: BackgroundTasks, db: SessionDep):
    order = await order_service.create(db, payload)
    bg.add_task(send_order_webhook_async, order.id)
    return order


async def send_order_webhook_async(order_id: int) -> None:
    async with httpx.AsyncClient(timeout=10.0) as client:
        # ... HMAC + POST
        pass
```

---

## 6. Subscription endpoint

(Flask 변형 [[../../python-flask/api-design/webhook-send#6 Subscription 관리|§6]] 와 동일)

---

## 7. Dead Letter 알림

(Flask 변형 §7 동일 — Celery beat)

---

## 9. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/webhook-send#9 함정 모음|§9]] + Flask 변형 함정)

### FastAPI 한정

### 함정 11 — async FastAPI + sync Celery worker 의 모델 공유
sync session + async session 양쪽 같은 모델 — `Base.metadata` 동일 + transaction 분리. **별도 session factory** (`AsyncSessionLocal`, `SyncSessionLocal`).

### 함정 12 — `httpx.AsyncClient` vs `httpx.Client`
FastAPI endpoint 안 = async (`AsyncClient`). Celery worker 안 = sync (`Client`). **혼동 X**.

### 함정 13 — BackgroundTasks 가 fail 시 silent
log / Sentry 보고. 중요한 webhook 은 **Celery + retry**.

---

## 10. 운영 체크리스트

(Java 원본 §10 + Python)

---

## 11. 관련

- [[../../spring/java-spring/api-design/webhook-send|↗ Java 원본]]
- [[../../python-flask/api-design/webhook-send|↗ Flask 변형]]
- [[distributed-lock]]
- [[../common/security-config]]
- [[api-design|↑ hub]]
