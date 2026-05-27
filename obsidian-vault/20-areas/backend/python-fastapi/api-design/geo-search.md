---
title: "위치 기반 검색 — FastAPI (PostGIS / Redis Geo)"
kind: knowledge
status: current
created_at: 2026-05-27T00:25:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, api-design, recipe, geo, postgis]
---

# 위치 기반 검색 — FastAPI (PostGIS / Redis Geo)

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/geo-search|↗ Java 원본]]** · **[[../../python-flask/api-design/geo-search|↗ Flask 변형]]**

> GeoAlchemy2 async + redis.asyncio GEO.

---

## 1-3-7. 무엇을 / DB 스키마 / privacy

(Java 원본 [[../../spring/java-spring/api-design/geo-search#1|§1, §2, §3]] 와 동일)

---

## 3-4. 반경 검색 (async)

```python
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession


async def find_nearby_items(
    db: AsyncSession,
    lat: float, lng: float, radius_m: int, limit: int,
) -> list[dict]:
    sql = text("""
        SELECT i.id AS id, i.title AS title, i.price_krw AS priceKrw,
               ST_Distance(i.coordinate,
                 ST_GeogFromText('POINT(' || :lng || ' ' || :lat || ')')) AS distanceM
        FROM items i
        WHERE i.status = 'SELLING'
          AND ST_DWithin(i.coordinate,
                ST_GeogFromText('POINT(' || :lng || ' ' || :lat || ')'), :radius_m)
        ORDER BY i.coordinate <->
                 ST_GeogFromText('POINT(' || :lng || ' ' || :lat || ')')
        LIMIT :limit
    """)
    result = await db.execute(sql, {
        "lat": lat, "lng": lng, "radius_m": radius_m, "limit": limit,
    })
    return [dict(r) for r in result.mappings().all()]
```

---

## 5. Redis Geo (async)

```python
from redis import asyncio as aioredis


async def nearby_items_redis(
    redis: aioredis.Redis,
    lng: float, lat: float, radius_km: float, limit: int,
) -> list[dict]:
    results = await redis.geosearch(
        "items:geo",
        longitude=lng, latitude=lat,
        radius=radius_km, unit="km",
        withcoord=True, withdist=True,
        sort="ASC", count=limit,
    )
    return [
        {"id": r[0], "distance_km": r[1], "lng": r[2][0], "lat": r[2][1]}
        for r in results
    ]


async def add_item_to_redis_geo(redis: aioredis.Redis, item_id: str, lng: float, lat: float) -> None:
    await redis.geoadd("items:geo", (lng, lat, item_id))
```

---

## 6. Controller — pydantic Field validation

```python
from pydantic import BaseModel, Field


class NearbyQuery(BaseModel):
    lat: float = Field(ge=33, le=43)
    lng: float = Field(ge=124, le=132)
    radius_m: int = Field(default=5000, ge=100, le=20000)
    limit: int = Field(default=20, ge=1, le=100)


@router.get("/items/nearby", response_model=CommonResponse[list[ItemNearby]])
async def nearby(
    db: SessionDep,
    query: NearbyQuery = Depends(),         # query string → pydantic
):
    items = await find_nearby_items(db, query.lat, query.lng, query.radius_m, query.limit)
    return CommonResponse.success(ResponseCode.OK, "조회 성공", items)
```

> **FastAPI idiom**: `Depends(NearbyQuery)` 가 query string 을 pydantic model 로 자동 검증.

---

## 7. distance label

```python
def distance_label(distance_m: float) -> str:
    if distance_m < 100:
        return "100m 이내"
    if distance_m < 1000:
        return f"{round(distance_m / 100) * 100}m"
    return f"{distance_m / 1000:.1f}km"
```

---

## 8. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/geo-search#9 함정 모음|§9]] + Flask 변형 함정)

### FastAPI 한정

### 함정 11 — `Depends(NearbyQuery)` vs `query: NearbyQuery = Depends()`
두 형식 모두 OK 지만 `Annotated` 권장:
```python
query: Annotated[NearbyQuery, Depends()]
```

### 함정 12 — `redis.asyncio.geosearch` arg 이름
redis-py 5.x sync 와 async API 동일하지만 일부 옵션명 다름 — docs 확인.

---

## 9. 운영 체크리스트

(Java 원본 §10 + Python async)

---

## 10. 관련

- [[../../spring/java-spring/api-design/geo-search|↗ Java 원본]]
- [[../../python-flask/api-design/geo-search|↗ Flask 변형]]
- [[cache-redis]]
- [[api-design|↑ hub]]
