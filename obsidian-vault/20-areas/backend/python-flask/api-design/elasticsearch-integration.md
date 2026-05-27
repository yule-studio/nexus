---
title: "Elasticsearch 통합 (한국어 nori / 색인 동기화) — Flask"
kind: knowledge
status: current
created_at: 2026-05-26T17:35:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-flask, api-design, recipe, elasticsearch]
---

# Elasticsearch 통합 (한국어 nori / 색인 동기화) — Flask

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/elasticsearch-integration|↗ Java 원본]]**

> `elasticsearch-py` 8.x 기반.

---

## 1-4. 언제 ES / 구성 / mapping JSON

(Java 원본 [[../../spring/java-spring/api-design/elasticsearch-integration#1|§1, §2, §4]] 와 동일 — mapping / nori 설정 언어 무관)

---

## 3. 의존성

```python
pip install elasticsearch>=8.13 celery[redis]
```

```python
# app/common/es.py
from elasticsearch import Elasticsearch
from app.common.config import settings

es_client = Elasticsearch(
    hosts=[settings.ES_URI],
    basic_auth=(settings.ES_USER, settings.ES_PASSWORD) if settings.ES_USER else None,
    request_timeout=5,
)
```

---

## 5. Outbox + Celery worker

```python
# app/repositories/search_outbox.py (SQLAlchemy)
class SearchIndexOutbox(Base):
    __tablename__ = "search_index_outbox"
    id = Column(String(26), primary_key=True)
    target_type = Column(String(30), nullable=False)
    target_id = Column(String(26), nullable=False)
    operation = Column(String(20), nullable=False)         # UPSERT / DELETE
    payload = Column(JSONB)
    status = Column(String(20), nullable=False, default="PENDING")
    attempts = Column(Integer, default=0)
    created_at = Column(DateTime(timezone=True), default=lambda: datetime.now(timezone.utc))
    processed_at = Column(DateTime(timezone=True))


# enqueue
def enqueue_search_index(target_type: str, target_id: str, operation: str = "UPSERT") -> None:
    db.session.add(SearchIndexOutbox(
        id=ulid_gen(),
        target_type=target_type,
        target_id=target_id,
        operation=operation,
        status="PENDING",
        attempts=0,
    ))


# domain event listener — SQLAlchemy before_commit
@event.listens_for(db.session, "before_commit")
def _enqueue_on_product_change(session):
    for obj in session.new:
        if isinstance(obj, Product):
            enqueue_search_index("PRODUCT", obj.id, "UPSERT")
```

```python
# app/workers/search_worker.py — Celery task
from celery import shared_task
from app.common.es import es_client
from app.repositories.search_outbox import SearchIndexOutbox


@shared_task(bind=True, max_retries=3, default_retry_delay=60)
def process_search_outbox(self):
    pending = db.session.query(SearchIndexOutbox).filter(
        SearchIndexOutbox.status == "PENDING"
    ).limit(100).all()

    for row in pending:
        try:
            if row.operation == "UPSERT":
                product = product_repo.find_by_id(row.target_id)
                if product is None:
                    raise ValueError(f"deleted: {row.target_id}")
                es_client.index(index="products_v1", id=product.id, document=_build_doc(product))
            elif row.operation == "DELETE":
                es_client.delete(index="products_v1", id=row.target_id, ignore=[404])

            row.status = "PROCESSED"
            row.processed_at = datetime.now(timezone.utc)
        except Exception as e:
            row.attempts += 1
            row.status = "FAILED" if row.attempts >= 5 else "PENDING"
            row.last_error = str(e)[:500]
    db.session.commit()
```

> **Python 한정**: Celery `bind=True` + `self.retry(countdown=N)` 로 명시적 재시도. SQLAlchemy session 이 task scope.

---

## 6. 검색 query

```python
def search_products(q: str, brand_ids: list[str] | None, price_min: int | None,
                    price_max: int | None, limit: int, cursor: list | None) -> dict:
    body = {"query": {"bool": {"filter": [{"term": {"status": "ACTIVE"}}]}}}

    if q:
        body["query"]["bool"]["must"] = [{
            "multi_match": {
                "query": q,
                "fields": ["name^3", "brand.name^2", "tags"],
                "type": "best_fields",
                "fuzziness": "AUTO",
            },
        }]
    if brand_ids:
        body["query"]["bool"]["filter"].append({"terms": {"brand.id": brand_ids}})
    if price_min is not None or price_max is not None:
        range_q = {"range": {"basePriceKrw": {}}}
        if price_min is not None:
            range_q["range"]["basePriceKrw"]["gte"] = price_min
        if price_max is not None:
            range_q["range"]["basePriceKrw"]["lte"] = price_max
        body["query"]["bool"]["filter"].append(range_q)

    body["sort"] = [{"createdAt": "desc"}, {"id": "desc"}]
    body["size"] = limit
    if cursor:
        body["search_after"] = cursor

    try:
        response = es_client.search(index="products_v1", body=body)
    except Exception as e:
        raise BusinessException(ResponseCode.EXTERNAL_API_ERROR, f"검색 실패: {e}")

    return _map_to_result(response)
```

---

## 7. 자동완성

```python
def autocomplete(prefix: str, limit: int) -> list[str]:
    response = es_client.search(
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

## 8. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/elasticsearch-integration#9 함정 모음|§9]] 의 10 함정 그대로)

### Python 한정

### 함정 11 — `elasticsearch-py` 7.x vs 8.x API 변화
`es.index(body=...)` (7.x) vs `es.index(document=...)` (8.x). **버전 명시 + lock**.

### 함정 12 — Celery worker session 누수
Flask-SQLAlchemy 의 `db.session` 이 Celery worker 에서 thread-local. `app.app_context()` 안에서 task 실행 (`celery_app.conf.task_default_queue` + `worker_init`).

---

## 9. 운영 체크리스트

(Java 원본 §10 그대로 + Python: elasticsearch-py 버전 lock / Celery session 관리)

---

## 10. 관련

- [[../../spring/java-spring/api-design/elasticsearch-integration|↗ Java 원본]]
- [[webhook-send]] — outbox 패턴 동일
- [[api-design|↑ hub]]
