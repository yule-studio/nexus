---
title: "Redis 캐시 (cache-aside + stampede 방지) — Flask"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T17:05:00+09:00
tags: [backend, python-flask, api-design, recipe, cache, redis]
---

# Redis 캐시 (cache-aside + stampede 방지) — Flask

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/cache-redis|↗ Java 원본]]**

> Flask-Caching + redis-py 기반. 외부 계약 (cache-aside 패턴 / TTL 정책) 은 Java 원본과 동일.

---

## 1. 캐시 패턴 4가지 / 2. 무엇을 캐시할까 / 7. 무효화 전략 / 9. key 설계

(Java 원본 [[../../spring/java-spring/api-design/cache-redis#1|§1, §2, §7, §9]] 와 동일 — 정책은 언어 무관)

---

## 3. 설정 — Flask-Caching

```python
# requirements
pip install Flask-Caching redis
```

```python
# app/__init__.py
from flask import Flask
from flask_caching import Cache

cache = Cache()


def create_app() -> Flask:
    app = Flask(__name__)
    app.config.from_mapping(
        CACHE_TYPE="RedisCache",
        CACHE_REDIS_URL="redis://localhost:6379/0",
        CACHE_DEFAULT_TIMEOUT=600,                       # 10 min default
        CACHE_KEY_PREFIX="shop:cache:",
    )
    cache.init_app(app)
    return app
```

---

## 4. `@cache.cached` / `@cache.memoize` 사용

```python
from flask_caching import Cache
from app import cache


class ProductQueryService:
    @cache.memoize(timeout=600, make_name=lambda fn: f"product:detail:{fn.__name__}")
    def get_detail(self, product_id: str) -> ProductDetailDto:
        product = product_repo.find_by_id(product_id)
        if product is None:
            raise BusinessException(ResponseCode.NOT_FOUND, "상품")
        return ProductDetailDto.from_entity(product)

    def evict(self, product_id: str) -> None:
        cache.delete_memoized(self.get_detail, product_id)
```

> **함정**: `@cache.memoize` 가 method 의 `self` 까지 hash 함 → 같은 instance 보장 필요. **stateless service** 또는 `cache.cached(key_prefix="product:detail")` 사용.

### 4.1 트랜잭션 안 evict 함정

```python
# ❌ 트랜잭션 rollback 시 캐시 사라짐
with db.session.begin():
    update_product(product_id)
    cache.delete_memoized(self.get_detail, product_id)

# ✅ AFTER_COMMIT 패턴 — SQLAlchemy event
from sqlalchemy import event

@event.listens_for(db.session, "after_commit")
def evict_after_commit(session):
    for pid in getattr(session, "_evict_product_ids", set()):
        cache.delete_memoized(ProductQueryService.get_detail, pid)
```

---

## 5. 직접 redis-py 사용

```python
from redis import Redis
from typing import Optional
import json

redis_client = Redis(host="localhost", port=6379, db=0, decode_responses=True)
TTL_SECONDS = 600


class ProductCache:
    def get(self, product_id: str) -> Optional[ProductDetailDto]:
        raw = redis_client.get(f"shop:cache:product:detail:{product_id}")
        return ProductDetailDto(**json.loads(raw)) if raw else None

    def set(self, product_id: str, value: ProductDetailDto) -> None:
        redis_client.setex(
            f"shop:cache:product:detail:{product_id}",
            TTL_SECONDS,
            json.dumps(value.__dict__, default=str),
        )

    def evict(self, product_id: str) -> None:
        redis_client.delete(f"shop:cache:product:detail:{product_id}")
```

---

## 6. Cache Stampede 방지

```python
import time
from redis.lock import Lock


def get_with_singleflight(product_id: str) -> ProductDetailDto:
    cached = product_cache.get(product_id)
    if cached is not None:
        return cached

    # singleflight — Redis lock
    with redis_client.lock(
        f"lock:cache-refresh:product:{product_id}",
        timeout=3,
        blocking_timeout=1,
    ) as lock:
        # 락 잡았으면 다시 캐시 확인 (다른 worker 가 채웠을 수도)
        cached = product_cache.get(product_id)
        if cached is not None:
            return cached

        fresh = _load_from_db(product_id)
        product_cache.set(product_id, fresh)
        return fresh
```

> **Python 한정 함정**: `redis.lock` 의 `blocking_timeout` 가 0 이면 무한 대기. `acquire()` 실패 시 `LockError` — fallback (DB 직접 / 다른 worker 가 채울 때까지 sleep) 처리 필요.

---

## 10. 측정 / 모니터링

```python
from prometheus_flask_exporter import PrometheusMetrics

metrics = PrometheusMetrics(app)
# cache hit / miss 는 custom counter

cache_hit = metrics.counter("cache_hit_total", "Cache hit count", labels={"key_prefix": lambda: ""})
cache_miss = metrics.counter("cache_miss_total", "Cache miss count")
```

---

## 11. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/cache-redis#11 함정 모음|§11]] 의 10 함정 그대로)

### Python/Flask 한정 추가

### 함정 11 — `@cache.memoize` 의 `self` hash
같은 method 라도 instance 가 다르면 다른 cache key. **module-level function** 으로 분리 또는 `@staticmethod`.

### 함정 12 — `redis.Redis(decode_responses=True)` vs `False`
True = str 반환, False = bytes 반환. **프로젝트 전체 통일** + JSON 직렬화 시 일관.

### 함정 13 — Flask multi-worker (gunicorn) 가 같은 cache 공유
`CACHE_TYPE="SimpleCache"` 는 process-local — multi-worker 면 동기화 안 됨. **`RedisCache`** 만.

---

## 12. 운영 체크리스트

- [ ] `CACHE_TYPE="RedisCache"` (SimpleCache X)
- [ ] hit/miss prometheus counter
- [ ] `maxmemory-policy allkeys-lru`
- [ ] stampede 방어 (singleflight Redis lock)
- [ ] evict 는 SQLAlchemy after_commit event
- [ ] 큰 객체 캐시 자제

---

## 13. 관련

- [[distributed-lock]] — stampede 에 사용
- [[../../spring/java-spring/api-design/cache-redis|↗ Java 원본]]
- [[api-design|↑ hub]]
