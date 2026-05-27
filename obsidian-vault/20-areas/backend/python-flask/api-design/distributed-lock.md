---
title: "분산 락 (redis-py / Redlock) — Flask"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T17:10:00+09:00
tags: [backend, python-flask, api-design, recipe, distributed-lock]
---

# 분산 락 (redis-py / Redlock) — Flask

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/distributed-lock|↗ Java 원본]]**

> redis-py 의 내장 `Lock` (single Redis) 또는 `redlock-py` (Redis Cluster) 기반.

---

## 1. JVM 락 vs 분산 / 2. 사용 시나리오 / 6. Fencing Token / 7. TTL 함정

(Java 원본 [[../../spring/java-spring/api-design/distributed-lock#1|§1, §2, §6, §7]] 와 동일)

---

## 3. 의존성

```python
pip install redis
# 분산 cluster 면
pip install redlock-py
```

---

## 4. 기본 — redis-py 의 `Lock`

```python
from redis import Redis
from redis.exceptions import LockError, LockNotOwnedError
from contextlib import contextmanager

redis_client = Redis(decode_responses=True)


@contextmanager
def distributed_lock(key: str, timeout: int = 5, blocking_timeout: int = 3):
    """Java 의 try-with-resources + Kotlin extension 동등."""
    lock = redis_client.lock(key, timeout=timeout, blocking_timeout=blocking_timeout)
    acquired = False
    try:
        acquired = lock.acquire()
        if not acquired:
            raise BusinessException(
                ResponseCode.RATE_LIMIT_EXCEEDED,
                f"락 획득 실패: {key}",
            )
        yield lock
    finally:
        if acquired:
            try:
                lock.release()
            except LockNotOwnedError:
                # TTL 만료 후 다른 worker 가 잡음 — 무시
                pass


# 사용
def decrease_stock(option_id: str, qty: int) -> None:
    with distributed_lock(f"lock:option:{option_id}", timeout=5, blocking_timeout=3):
        with db.session.begin():
            option = option_repo.find_by_id(option_id)
            if option is None:
                raise BusinessException(ResponseCode.NOT_FOUND)
            option.decrease_stock(qty)
```

> **Python idiom**: `@contextmanager` + `yield` 로 `try-finally` + `release` 보장. Java try-with-resources 동등.

---

## 5. Decorator 식

```python
from functools import wraps
from typing import Callable


def with_lock(key_fn: Callable[..., str], timeout: int = 5, blocking_timeout: int = 3):
    """Spring @DistributedLock 동등."""
    def decorator(fn):
        @wraps(fn)
        def wrapper(*args, **kwargs):
            key = key_fn(*args, **kwargs)
            with distributed_lock(f"lock:{key}", timeout=timeout, blocking_timeout=blocking_timeout):
                return fn(*args, **kwargs)
        return wrapper
    return decorator


# 사용
@with_lock(key_fn=lambda option_id, qty: f"option:{option_id}", timeout=5)
def decrease_stock(option_id: str, qty: int) -> None:
    # ...
    pass
```

> **Python 한정**: `key_fn` 가 lambda — SpEL 보다 직접적이고 typed. parameter name 이 함수 signature 와 동일해야 inspect 가 가능.

---

## 8. `@scheduled` 분산 — Celery beat + redbeat / ShedLock 동등

Celery beat 가 multi-instance 일 때, 같은 task 가 여러 worker 에 실행되는 것 방지:

```python
# requirements
pip install celery-redbeat

# celery_app.py
from celery import Celery

app = Celery("shop", broker="redis://localhost:6379/0")
app.conf.beat_scheduler = "redbeat.RedBeatScheduler"
app.conf.redbeat_redis_url = "redis://localhost:6379/0"
app.conf.beat_schedule = {
    "expire-orders": {
        "task": "app.workers.orders.expire_orders",
        "schedule": 60.0,
    },
}
```

→ **redbeat** 가 Redis 분산 락으로 single execution 보장.

---

## 9. 테스트

```python
import concurrent.futures
from threading import Event


def test_concurrent_stock_decrease_does_not_oversell():
    option = create_option_fixture(stock=50)
    success_count = 0
    lock_obj = threading.Lock()

    def attempt():
        nonlocal success_count
        try:
            decrease_stock(option.id, 1)
            with lock_obj:
                success_count += 1
        except BusinessException:
            pass

    with concurrent.futures.ThreadPoolExecutor(max_workers=20) as executor:
        futures = [executor.submit(attempt) for _ in range(100)]
        concurrent.futures.wait(futures)

    assert success_count == 50
    assert reload_option(option.id).stock == 0
```

---

## 10. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/distributed-lock#10 함정 모음|§10]] 의 10 함정 그대로)

### Python 한정

### 함정 11 — `redis-py` lock 의 default `blocking=True`
acquire() 가 timeout 까지 blocking. `blocking=False` 또는 `blocking_timeout` 명시.

### 함정 12 — gunicorn `--preload` 와 redis connection
preload 시 master process 에서 connection 생성 → fork 후 자식이 같은 connection 공유 = race condition. **lazy init** 또는 `--preload` 없이.

### 함정 13 — Celery worker 의 lock release
worker 가 kill -9 받으면 `finally` 안 돔 → TTL 만 의존. **짧은 TTL + idempotent task**.

---

## 11. 운영 체크리스트

(Java 원본 §11 그대로)

- [ ] `redis.lock` 의 timeout = 작업 예상 × 3~5
- [ ] `@contextmanager` 로 release 보장
- [ ] Celery beat 는 `redbeat` 등 분산 backend
- [ ] gunicorn `--preload` + redis 호환 검토

---

## 12. 관련

- [[cache-redis]] — stampede 방어
- [[../../spring/java-spring/api-design/distributed-lock|↗ Java 원본]]
- [[api-design|↑ hub]]
