---
title: "Elasticsearch 통합 — FastAPI"
kind: knowledge
status: current
created_at: 2026-05-27T00:05:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, api-design, recipe, elasticsearch]
---

# Elasticsearch 통합 — FastAPI

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/elasticsearch-integration|↗ Java 원본]]** · **[[../../python-flask/api-design/elasticsearch-integration|↗ Flask 변형]]**

> `elasticsearch[async]` 8.x async client.

---

## 1-2-4-7-8. 언제 ES / 구성 / mapping / 자동완성 / 인덱스 마이그레이션

(Java 원본 [[../../spring/java-spring/api-design/elasticsearch-integration#1|§1, §2, §4, §7, §8]] 와 동일)

---

## 3. 의존성

```bash
pip install "elasticsearch[async]>=8.13"
```

```python
# app/extensions/es.py
from elasticsearch import AsyncElasticsearch
from app.config import settings

es_client = AsyncElasticsearch(
    hosts=[settings.ES_URI],
    basic_auth=(settings.ES_USER, settings.ES_PASSWORD) if settings.ES_USER else None,
    request_timeout=5.0,
)


# lifespan close
async def close_es():
    await es_client.close()
```

---

## 5. Outbox + Celery 또는 FastAPI BackgroundTasks

```python
# app/services/search_outbox.py
from datetime import datetime, timezone
from sqlalchemy.ext.asyncio import AsyncSession
from app.domain.search_outbox import SearchIndexOutbox


async def enqueue_search_index(
    db: AsyncSession, target_type: str, target_id: str, operation: str = "UPSERT",
) -> None:
    db.add(SearchIndexOutbox(
        target_type=target_type,
        target_id=target_id,
        operation=operation,
        status="PENDING",
    ))
    # commit 은 호출자가


# Celery task (장기 처리) 또는 별도 async worker
async def process_search_outbox_batch(db: AsyncSession) -> None:
    pending = await db.scalars(
        select(SearchIndexOutbox).where(SearchIndexOutbox.status == "PENDING").limit(100)
    )
    for row in pending.all():
        try:
            if row.operation == "UPSERT":
                product = await db.get(Product, row.target_id)
                if product is None:
                    raise ValueError(f"deleted: {row.target_id}")
                await es_client.index(
                    index="products_v1",
                    id=product.id,
                    document=_build_doc(product),
                )
            elif row.operation == "DELETE":
                await es_client.delete(
                    index="products_v1", id=row.target_id, ignore=[404]
                )
            row.status = "PROCESSED"
            row.processed_at = datetime.now(timezone.utc)
        except Exception as e:
            row.attempts += 1
            row.status = "FAILED" if row.attempts >= 5 else "PENDING"
            row.last_error = str(e)[:500]
    await db.commit()
```

---

## 6. 검색 query (async)

```python
async def search_products(
    q: str, brand_ids: list[str] | None, price_min: int | None,
    price_max: int | None, limit: int, cursor: list | None,
) -> dict:
    body: dict = {
        "query": {"bool": {"filter": [{"term": {"status": "ACTIVE"}}]}},
        "size": limit,
        "sort": [{"createdAt": "desc"}, {"id": "desc"}],
    }
    bool_query = body["query"]["bool"]

    if q:
        bool_query["must"] = [{
            "multi_match": {
                "query": q,
                "fields": ["name^3", "brand.name^2", "tags"],
                "type": "best_fields",
                "fuzziness": "AUTO",
            },
        }]
    if brand_ids:
        bool_query["filter"].append({"terms": {"brand.id": brand_ids}})
    if price_min is not None or price_max is not None:
        rng: dict = {"range": {"basePriceKrw": {}}}
        if price_min is not None:
            rng["range"]["basePriceKrw"]["gte"] = price_min
        if price_max is not None:
            rng["range"]["basePriceKrw"]["lte"] = price_max
        bool_query["filter"].append(rng)
    if cursor:
        body["search_after"] = cursor

    try:
        response = await es_client.search(index="products_v1", body=body)
    except Exception as e:
        raise BusinessException(503, "EXTERNAL_API_ERROR", f"검색 실패: {e}")

    return _map_to_result(response)


async def autocomplete(prefix: str, limit: int) -> list[str]:
    response = await es_client.search(
        index="products_v1",
        body={
            "query": {"match": {"name.auto": prefix}},
            "size": limit,
            "_source": ["name"],
        },
    )
    return list({hit["_source"]["name"] for hit in response["hits"]["hits"]})
```

---

## 9. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/elasticsearch-integration#9 함정 모음|§9]] 의 10 함정 그대로)

### FastAPI 한정

### 함정 11 — sync `Elasticsearch` 와 async `AsyncElasticsearch` 혼용
async endpoint 에서 sync client = event loop block. **`AsyncElasticsearch`** 만.

### 함정 12 — `await es_client.close()` lifespan 누락
shutdown 시 connection 누수. **`@asynccontextmanager lifespan`** 안에서 close.

### 함정 13 — `ignore=[404]` 가 ES 8.x 에서 deprecated
`raise_on_error=False` 또는 try/except 처리 권장.

---

## 10. 운영 체크리스트

(Java 원본 §10 + Python async)

---

## 11. 관련

- [[../../spring/java-spring/api-design/elasticsearch-integration|↗ Java 원본]]
- [[../../python-flask/api-design/elasticsearch-integration|↗ Flask 변형]]
- [[webhook-send]] — outbox 패턴 동일
- [[api-design|↑ hub]]
