---
title: "Webhook 송신 (서명 / 재시도 / DLQ) — Flask"
kind: knowledge
status: current
created_at: 2026-05-26T22:10:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-flask, api-design, recipe, webhook]
---

# Webhook 송신 (서명 / 재시도 / DLQ) — Flask

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/webhook-send|↗ Java 원본]]**

> Outbox + Celery worker + HMAC-SHA-256 서명.

---

## 1. 무엇을 / 2. DB / 8. 수신자 가이드

(Java 원본 [[../../spring/java-spring/api-design/webhook-send#1|§1, §2, §8]] 와 동일)

---

## 3. Outbox enqueue (SQLAlchemy)

```python
# app/services/webhook_service.py
from app.domain.webhook import WebhookSubscription, WebhookDelivery, DeliveryStatus
from app.extensions import db


def enqueue_webhook(event_type: str, event_id: str, payload: dict) -> None:
    """같은 트랜잭션에서 호출. domain event listener 또는 service 끝."""
    subs = (
        db.session.query(WebhookSubscription)
        .filter(WebhookSubscription.active.is_(True))
        .filter(WebhookSubscription.event_types.contains(event_type))
        .all()
    )
    for sub in subs:
        try:
            db.session.add(WebhookDelivery(
                subscription_id=sub.id,
                event_id=event_id,
                event_type=event_type,
                payload=payload,
                status=DeliveryStatus.PENDING.value,
                attempts=0,
                next_attempt_at=datetime.now(timezone.utc),
            ))
            db.session.flush()
        except IntegrityError:
            db.session.rollback()           # (sub_id, event_id) UNIQUE — 멱등
```

---

## 4. Worker — Celery

```python
# app/workers/webhook_worker.py
import hmac
import hashlib
import time
import json
import httpx

from celery import shared_task
from app.domain.webhook import WebhookDelivery, WebhookSubscription, DeliveryStatus
from app.extensions import db

# Exponential backoff: 30s → 5m → 30m → 2h → 6h → 12h → 24h → DEAD_LETTER
_BACKOFF_SECONDS = [30, 300, 1800, 7200, 21600, 43200, 86400]


@shared_task(bind=True)
def process_webhook_deliveries(self) -> None:
    """매 1초 — Celery beat 가 schedule."""
    due = (
        db.session.query(WebhookDelivery)
        .filter(WebhookDelivery.status.in_([DeliveryStatus.PENDING.value, DeliveryStatus.RETRY.value]))
        .filter(WebhookDelivery.next_attempt_at <= datetime.now(timezone.utc))
        .limit(50)
        .all()
    )
    for d in due:
        deliver_one.delay(d.id)


@shared_task(bind=True)
def deliver_one(self, delivery_id: int) -> None:
    delivery = db.session.get(WebhookDelivery, delivery_id)
    if delivery is None or delivery.status not in (
        DeliveryStatus.PENDING.value, DeliveryStatus.RETRY.value
    ):
        return

    sub = db.session.get(WebhookSubscription, delivery.subscription_id)
    if sub is None or not sub.active:
        delivery.status = DeliveryStatus.DEAD_LETTER.value
        delivery.last_error = "subscription inactive"
        db.session.commit()
        return

    payload_str = json.dumps(delivery.payload, separators=(",", ":"), default=str)
    signature = hmac.new(
        sub.secret.encode("utf-8"),
        payload_str.encode("utf-8"),
        hashlib.sha256,
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
            delivery.last_response_status = response.status_code
        elif 400 <= response.status_code < 500:
            # 4xx — 영구 실패 (재시도 무의미)
            delivery.status = DeliveryStatus.DEAD_LETTER.value
            delivery.last_response_status = response.status_code
            delivery.last_error = response.text[:500]
        else:
            # 5xx — retry
            _schedule_retry(delivery, response.status_code, response.text)
    except Exception as e:
        _schedule_retry(delivery, None, str(e))

    db.session.commit()


def _schedule_retry(delivery: WebhookDelivery, status: int | None, error: str | None) -> None:
    delivery.attempts += 1
    delivery.last_response_status = status
    delivery.last_error = (error or "")[:500]

    if delivery.attempts > len(_BACKOFF_SECONDS):
        delivery.status = DeliveryStatus.DEAD_LETTER.value
    else:
        delivery.status = DeliveryStatus.RETRY.value
        delivery.next_attempt_at = datetime.now(timezone.utc) + timedelta(
            seconds=_BACKOFF_SECONDS[delivery.attempts - 1],
        )
```

> **Python idiom 포인트**:
> - `hmac.new(key, msg, hashlib.sha256).hexdigest()` — Java `Mac.getInstance("HmacSHA256")` 보다 한 줄.
> - `httpx.Client(timeout=10.0)` — context manager 가 connection close.
> - `json.dumps(..., separators=(",", ":"))` — whitespace 최소 (서명 일관).

### 4.1 Celery beat — 1초마다 polling

```python
# celery_app.py
from celery import Celery
from celery.schedules import crontab

celery_app = Celery("shop", broker="redis://localhost:6379/0")
celery_app.conf.beat_schedule = {
    "process-webhook-deliveries": {
        "task": "app.workers.webhook_worker.process_webhook_deliveries",
        "schedule": 1.0,                      # 1 second
    },
}
```

---

## 5. HTTP Client — circuit breaker

`httpx` 자체엔 circuit breaker 없음. `pybreaker` 사용:

```python
# pip install pybreaker
import pybreaker

webhook_breaker = pybreaker.CircuitBreaker(
    fail_max=5,
    reset_timeout=60,
    name="webhook",
)


@webhook_breaker
def call_webhook(url: str, content: str, headers: dict) -> httpx.Response:
    with httpx.Client(timeout=10.0) as client:
        return client.post(url, content=content, headers=headers)
```

→ 5회 실패 시 60s 동안 즉시 fail (다른 subscriber 보호).

---

## 6. Subscription 관리 endpoint

```python
@webhook_bp.post("/subscriptions")
@role_required("SELLER", "ADMIN", "MASTER")
def create_subscription():
    payload = CreateSubSchema().load(request.get_json())
    sub = webhook_service.create(get_jwt_identity(), payload)
    # secret 은 한 번만 노출
    return ok(
        {"id": sub.id, "secret": sub.secret},
        message="구독 생성. secret 은 다시 노출되지 않습니다.",
        status=201,
    )


@webhook_bp.post("/subscriptions/<int:sub_id>/test")
@role_required("SELLER", "ADMIN", "MASTER")
def test_send(sub_id: int):
    webhook_service.send_test(sub_id, get_jwt_identity())
    return ok(message="테스트 이벤트 발송")
```

---

## 7. Dead Letter 알림

```python
@shared_task
def report_dead_letters() -> None:
    """매일 오전 9시 — Celery beat."""
    yesterday = datetime.now(timezone.utc) - timedelta(days=1)
    dead = db.session.query(WebhookDelivery).filter(
        WebhookDelivery.status == DeliveryStatus.DEAD_LETTER.value,
        WebhookDelivery.created_at >= yesterday,
    ).all()
    if not dead:
        return

    by_url = {}
    for d in dead:
        url = d.subscription.target_url
        by_url[url] = by_url.get(url, 0) + 1

    slack_send("Webhook DLQ alert", str(by_url))
```

---

## 9. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/webhook-send#9 함정 모음|§9]] 의 10 함정 그대로)

### Python/Flask 한정

### 함정 11 — `json.dumps` 의 separator 가 서명에 영향
default separator `, ` (whitespace 포함) vs `,` — 서명 다름. **`separators=(",", ":")` 고정** 또는 raw bytes 직접 처리.

### 함정 12 — `httpx.Client` connection pool 누수
context manager (`with httpx.Client(...) as c`) 사용. 글로벌 client 도 가능하지만 process 종료 시 `client.close()`.

### 함정 13 — Celery `@shared_task(bind=True)` 의 `self`
retry / metadata 접근 시 필요. `self.retry(countdown=N, exc=...)` 명시적 재시도 가능.

---

## 10. 운영 체크리스트

(Java 원본 §10 그대로 + Python: pybreaker / json separator / httpx client)

---

## 11. 관련

- [[../../spring/java-spring/api-design/webhook-send|↗ Java 원본]]
- [[distributed-lock]] — ShedLock 동등 (Celery beat + redbeat)
- [[../common/security-config]]
- [[api-design|↑ hub]]
