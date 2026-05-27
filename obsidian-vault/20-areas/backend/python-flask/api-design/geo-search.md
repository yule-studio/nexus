---
title: "위치 기반 검색 — Flask (PostGIS / Redis Geo)"
kind: knowledge
status: current
created_at: 2026-05-26T17:55:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-flask, api-design, recipe, geo, postgis]
---

# 위치 기반 검색 — Flask (PostGIS / Redis Geo)

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/geo-search|↗ Java 원본]]**

> `GeoAlchemy2` (SQLAlchemy + PostGIS) 또는 redis-py GEO.

---

## 1-3. 무엇을 / 도구 비교 / DB 스키마

(Java 원본 [[../../spring/java-spring/api-design/geo-search#1|§1, §2, §3]] 와 동일 — PostGIS DDL 언어 무관)

---

## 2. 의존성

```python
pip install GeoAlchemy2 Shapely redis
```

```python
# Entity 예시
from geoalchemy2 import Geography

class ItemEntity(Base):
    __tablename__ = "items"
    id = Column(String(26), primary_key=True)
    coordinate = Column(Geography(geometry_type="POINT", srid=4326), nullable=False)
```

---

## 3. 반경 검색 (ST_DWithin)

```python
from sqlalchemy import text


def find_nearby_items(lat: float, lng: float, radius_m: int, limit: int) -> list[dict]:
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
    rows = db.session.execute(sql, {
        "lat": lat, "lng": lng, "radius_m": radius_m, "limit": limit,
    }).mappings().all()
    return [dict(r) for r in rows]
```

---

## 4. 동네 인증

```python
def verify_neighborhood(user_id: str, lat: float, lng: float) -> dict:
    neighborhood = db.session.execute(
        text("""
            SELECT id, name, sido, sigungu FROM neighborhoods
            WHERE ST_Contains(
                boundary::geometry,
                ST_GeomFromText('POINT(' || :lng || ' ' || :lat || ')', 4326)
            )
            LIMIT 1
        """),
        {"lng": lng, "lat": lat},
    ).mappings().first()

    if neighborhood is None:
        raise BusinessException(ResponseCode.NOT_FOUND, "해당 위치의 동네를 찾을 수 없습니다.")

    # user_locations upsert
    user_location_repo.upsert(user_id, neighborhood["id"], lng, lat)
    return dict(neighborhood)
```

---

## 5. Redis Geo (옵션)

```python
from redis import Redis


def add_item_to_redis_geo(item_id: str, lng: float, lat: float) -> None:
    redis_client.geoadd("items:geo", (lng, lat, item_id))


def nearby_items_redis(lng: float, lat: float, radius_km: float, limit: int) -> list[dict]:
    results = redis_client.geosearch(
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
```

> **redis-py idiom**: `geosearch` (redis-py 4.0+) — 옛 `georadius` 는 deprecated.

---

## 6. Controller + 입력 검증

```python
from marshmallow import Schema, fields, validate


class NearbyQuerySchema(Schema):
    lat = fields.Float(required=True, validate=validate.Range(min=33, max=43))
    lng = fields.Float(required=True, validate=validate.Range(min=124, max=132))
    radius_m = fields.Integer(load_default=5000, validate=validate.Range(min=100, max=20000))
    limit = fields.Integer(load_default=20, validate=validate.Range(min=1, max=100))


@geo_bp.get("/items/nearby")
def nearby():
    args = NearbyQuerySchema().load(request.args)
    items = find_nearby_items(args["lat"], args["lng"], args["radius_m"], args["limit"])
    return ok("조회 성공", items)
```

---

## 7. privacy — 거리 라벨

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

(Java 원본 [[../../spring/java-spring/api-design/geo-search#9 함정 모음|§9]] 의 10 함정 그대로)

### Python 한정

### 함정 11 — `ST_DWithin` parameter binding
`:lng + 0.0001` 같은 산술 가능. 그러나 SQL 안에서 `||` 로 string 만들기는 lat/lng 순서 반드시 (lng lat) 명시.

### 함정 12 — `redis_client.geoadd` arg 순서
`(lng, lat, name)` — Java 의 `Point(lng, lat)` 와 동일하지만 redis-py 의 GEO 명령은 항상 longitude 먼저.

---

## 9. 운영 체크리스트

(Java 원본 §10 그대로)

---

## 10. 관련

- [[../../spring/java-spring/api-design/geo-search|↗ Java 원본]]
- [[cache-redis]]
- [[api-design|↑ hub]]
