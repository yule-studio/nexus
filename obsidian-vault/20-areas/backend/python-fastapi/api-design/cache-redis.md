---
title: "Redis 캐시 (cache-aside + stampede 방지) — FastAPI"
kind: knowledge
status: current
created_at: 2026-05-26T23:35:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, api-design, recipe, cache, redis]
---

# Redis 캐시 (cache-aside + stampede 방지) — FastAPI

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/cache-redis|↗ Java 원본]]** · **[[../../python-flask/api-design/cache-redis|↗ Flask 변형]]**

> `redis.asyncio` 기반 async cache. fastapi-cache 또는 직접.

---

## 1-2-7-9. 패턴 / 무엇을 캐시 / 무효화 / key 설계

(Java 원본 [[../../spring/java-spring/api-design/cache-redis#1|§1, §2, §7, §9]] 와 동일 — 정책은 언어 무관)

---

## 3. 의존성

```bash
pip install redis fastapi-cache2[redis]
```

```python
# app/extensions/cache.py
from redis import asyncio as aioredis
from fastapi_cache import FastAPICache
from fastapi_cache.backends.redis import RedisBackend

from app.config import settings


async def init_cache() -> None:
    redis_client = aioredis.from_url(
        settings.REDIS_URL,
        encoding="utf-8",
        decode_responses=True,
    )
    FastAPICache.init(RedisBackend(redis_client), prefix="shop:cache:")


# app/main.py 의 lifespan 에서
@asynccontextmanager
async def lifespan(app: FastAPI):
    await init_cache()
    yield
    await FastAPICache.clear()
```

---

## 4. `@cache` decorator 사용

```python
from fastapi_cache.decorator import cache


@router.get("/products/{product_id}", response_model=ProductDetailResponse)
@cache(expire=600, namespace="product:detail")
async def get_product(product_id: str, db: SessionDep) -> ProductDetail:
    product = await product_repo.find_by_id(db, product_id)
    if product is None:
        raise NotFoundException("상품")
    return product
```

> **Python/FastAPI 한정**: `@cache` 는 함수 인자 hash 기반 자동 key. `@cache(key_builder=...)` 로 custom 가능.

### 4.1 트랜잭션 안 evict 함정

```python
# ❌ DB rollback 시 캐시 이미 사라짐
async def update_product(db: AsyncSession, product_id: str, ...) -> None:
    product = await db.get(Product, product_id)
    product.name = new_name
    await db.commit()
    await FastAPICache.clear(namespace="product:detail")    # rollback 무관 즉시 삭제

# ✅ commit 성공 후만 evict
async def update_product(db: AsyncSession, product_id: str, ...) -> None:
    product = await db.get(Product, product_id)
    product.name = new_name
    await db.commit()
    # commit 성공 보장 후
    await _evict_product_cache(product_id)
```

SQLAlchemy async event 로는 자동 평가가 어려움 — 명시적 patten 권장.

---

## 5. 직접 redis.asyncio 사용

```python
# app/services/product_cache.py
from typing import Optional
import json
from redis import asyncio as aioredis

from app.schemas.product import ProductDetailResponse


class ProductCache:
    def __init__(self, redis: aioredis.Redis) -> None:
        self.redis = redis
        self.ttl_seconds = 600

    async def get(self, product_id: str) -> Optional[ProductDetailResponse]:
        raw = await self.redis.get(f"shop:cache:product:detail:{product_id}")
        return ProductDetailResponse(**json.loads(raw)) if raw else None

    async def set(self, product_id: str, value: ProductDetailResponse) -> None:
        await self.redis.setex(
            f"shop:cache:product:detail:{product_id}",
            self.ttl_seconds,
            value.model_dump_json(),
        )

    async def evict(self, product_id: str) -> None:
        await self.redis.delete(f"shop:cache:product:detail:{product_id}")
```

---

## 6. Cache Stampede 방지 — async lock

```python
async def get_with_singleflight(
    product_cache: ProductCache, redis: aioredis.Redis, product_id: str,
) -> ProductDetailResponse:
    cached = await product_cache.get(product_id)
    if cached:
        return cached

    # singleflight
    async with redis.lock(
        f"lock:cache-refresh:product:{product_id}",
        timeout=3,
        blocking_timeout=1,
    ):
        # 다시 확인 (다른 worker 가 채웠을 수도)
        cached = await product_cache.get(product_id)
        if cached:
            return cached

        fresh = await _load_from_db(product_id)
        await product_cache.set(product_id, fresh)
        return fresh
```

> **FastAPI idiom 포인트**:
> - `async with redis.lock(...)` — Python async context manager.
> - `await` 만으로 자연스러운 chain.

---

## 10. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/cache-redis#11 함정 모음|§11]] 의 함정 그대로)

### Python/FastAPI 한정

### 함정 11 — sync redis-py + async FastAPI 혼용
`redis.Redis(...)` (sync) 를 async endpoint 안에서 호출 = 이벤트 loop block. **`redis.asyncio.Redis`** 만.

### 함정 12 — `@cache` decorator + pydantic response model
pydantic 객체 직렬화 시 datetime / Decimal 등 직렬화 실패 → cache 시 `TypeError`. **`model_dump_mode="json"`** 또는 `default=str`.

### 함정 13 — `FastAPICache.init` 가 lifespan 외에서 호출
import 시점에 redis 연결 시도 → race. **`@asynccontextmanager lifespan`** 안에서만.

---

## 11. 운영 체크리스트

- [ ] redis.asyncio (sync redis-py X)
- [ ] FastAPICache lifespan init
- [ ] @cache decorator + namespace 명시
- [ ] commit 성공 후만 evict
- [ ] stampede 방어 (async lock)
- [ ] `maxmemory-policy allkeys-lru`

---

## 12. 관련

- [[../../spring/java-spring/api-design/cache-redis|↗ Java 원본]]
- [[../../python-flask/api-design/cache-redis|↗ Flask 변형]]
- [[distributed-lock]]
- [[api-design|↑ hub]]
