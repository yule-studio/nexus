---
title: "Elasticsearch 통합 (한국어 nori / 색인 동기화) — Kotlin"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T12:00:00+09:00
tags:
  - backend
  - kotlin-spring
  - api-design
  - recipe
  - elasticsearch
  - search
---

# Elasticsearch 통합 (한국어 nori / 색인 동기화) — Kotlin

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + Kotlin 빌더 lambda + 함정 |

**[[api-design|↑ api-design hub]]** · **[[../../java-spring/api-design/elasticsearch-integration|↗ Java 원본]]**

> 📐 공통: [[../common/response-envelope]]. PG → ES 전환 시점은 Java 원본 [[../../java-spring/api-design/elasticsearch-integration#1 언제 ES 로|§1]].

---

## 1. 언제 ES 로 / 2. 구성 / 4. Index Mapping / 7. 자동완성 / 8. 인덱스 마이그레이션

(Java 원본 §1, §2, §4, §7, §8 와 동일 — 인프라 / mapping JSON / nori 분석기는 언어와 무관)

---

## 3. 의존성

```kotlin
implementation("co.elastic.clients:elasticsearch-java:8.13.4")
// 또는 Spring Data Elasticsearch (간단하지만 8.x 호환 주의)
implementation("org.springframework.boot:spring-boot-starter-data-elasticsearch")
```

```yaml
spring:
  elasticsearch:
    uris: ${ES_URIS:http://localhost:9200}
    username: ${ES_USER:}
    password: ${ES_PASSWORD:}
    socket-timeout: 5s
    connection-timeout: 3s

app:
  search:
    index:
      product: products_v1                     # 버전 prefix
```

---

## 5. 색인 동기화 — Outbox 패턴 (권장)

```sql
CREATE TABLE search_index_outbox (
    id            CHAR(26) PRIMARY KEY,
    target_type   VARCHAR(30) NOT NULL,            -- PRODUCT / USER
    target_id     CHAR(26) NOT NULL,
    operation     VARCHAR(20) NOT NULL,             -- UPSERT / DELETE
    payload       JSONB,
    status        VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    attempts      INTEGER NOT NULL DEFAULT 0,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at  TIMESTAMPTZ
);
CREATE INDEX ix_search_outbox_pending ON search_index_outbox (status, created_at)
    WHERE status = 'PENDING';
```

```kotlin
@Service
class SearchIndexOutboxService(
    private val outbox: SearchIndexOutboxRepository,
    private val ids: IdGenerator,
    private val clock: Clock,
) {
    @Transactional(propagation = Propagation.MANDATORY)
    fun enqueue(targetType: String, targetId: String, operation: String, payload: Any? = null) {
        outbox.save(
            SearchIndexOutboxRow(
                id = ids.next(),
                targetType = targetType,
                targetId = targetId,
                operation = operation,
                payload = payload,
                status = "PENDING",
                attempts = 0,
                createdAt = Instant.now(clock),
                processedAt = null,
            ),
        )
    }
}

@Component
class ProductIndexEnqueuer(
    private val outbox: SearchIndexOutboxService,
) {
    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    fun on(event: ProductCreated) {
        outbox.enqueue("PRODUCT", event.productId.value, "UPSERT")
    }
    // ProductUpdated / ProductActivated / ProductDeleted 동일
}
```

```kotlin
// Worker
@Component
class SearchIndexWorker(
    private val outbox: SearchIndexOutboxRepository,
    private val es: ElasticsearchClient,
    private val products: ProductRepository,
) {
    private val log = LoggerFactory.getLogger(javaClass)

    @Scheduled(fixedDelay = 1000)
    @SchedulerLock(name = "searchIndexWorker", lockAtMostFor = "5m")
    fun process() {
        outbox.findPending(100).forEach { processOne(it.id) }
    }

    @Transactional
    fun processOne(id: SearchIndexOutboxId) {
        val row = outbox.findByIdForUpdate(id) ?: return
        if (row.status != "PENDING") return

        runCatching {
            when (row.operation) {
                "UPSERT" -> {
                    val product = products.findById(ProductId(row.targetId))
                        ?: error("deleted: ${row.targetId}")
                    val doc = buildDoc(product)
                    es.index { it.index("products_v1").id(product.id.value).document(doc) }
                }
                "DELETE" -> es.delete { it.index("products_v1").id(row.targetId) }
                else     -> log.warn("unknown operation: {}", row.operation)
            }
            row.markProcessed(Instant.now())
        }.onFailure { ex ->
            row.recordFailure(ex.message ?: ex.javaClass.simpleName)
        }
        outbox.save(row)
    }
}
```

> **Kotlin idiom 포인트**:
> - `es.index { it.index(...).id(...).document(doc) }` — Java fluent lambda `i -> i.index(...)` 가 Kotlin trailing lambda `{ it.x().y() }` 로.
> - `runCatching { } / .onFailure { }` — try/catch + log 한 chain.
> - `when` 의 exhaustive (else 누락 시 컴파일러 경고).

### 5.2 Debezium CDC / 5.3 동기 색인

(Java 원본 [[../../java-spring/api-design/elasticsearch-integration#5.2 Debezium CDC|§5.2]] / §5.3 와 동일)

---

## 6. 검색 query

```kotlin
@Service
class ProductSearchService(
    private val es: ElasticsearchClient,
) {

    fun search(c: ProductSearchCriteria, limit: Int, cursor: Cursor?): SearchResult {
        val bool = QueryBuilders.bool().apply {
            // status filter
            filter { it.term { t -> t.field("status").value("ACTIVE") } }

            // 키워드 검색 (name + brand)
            if (!c.q.isNullOrBlank()) {
                must { m ->
                    m.multiMatch { mm ->
                        mm.query(c.q)
                            .fields("name^3", "brand.name^2", "tags")
                            .type(TextQueryType.BestFields)
                            .fuzziness("AUTO")
                    }
                }
            }

            // 브랜드 (다중)
            if (!c.brand.isNullOrEmpty()) {
                filter { f ->
                    f.terms { t ->
                        t.field("brand.id").terms { tv ->
                            tv.value(c.brand.map(FieldValue::of))
                        }
                    }
                }
            }

            // 카테고리 (path prefix)
            c.categoryPath?.let { path ->
                filter { it.prefix { p -> p.field("categoryPath").value(path) } }
            }

            // 가격 range
            if (c.priceMin != null || c.priceMax != null) {
                filter { f ->
                    f.range { r ->
                        r.field("basePriceKrw").also {
                            c.priceMin?.let { v -> it.gte(JsonData.of(v)) }
                            c.priceMax?.let { v -> it.lte(JsonData.of(v)) }
                        }
                    }
                }
            }
        }

        val sort: SortOptions = when (c.sort) {
            SortMode.LATEST    -> SortOptions.of { it.field { f -> f.field("createdAt").order(SortOrder.Desc) } }
            SortMode.PRICE_ASC -> SortOptions.of { it.field { f -> f.field("basePriceKrw").order(SortOrder.Asc) } }
            SortMode.POPULAR   -> SortOptions.of { it.field { f -> f.field("saleCount30d").order(SortOrder.Desc) } }
            else               -> SortOptions.of { it.score { sc -> sc.order(SortOrder.Desc) } }
        }

        return runCatching {
            val response = es.search({ s ->
                s.index("products_v1")
                    .query(bool.build()._toQuery())
                    .sort(sort)
                    .size(limit)
                    .searchAfter(cursor?.let { listOf(FieldValue.of(it.lastSort)) })
            }, ProductDoc::class.java)
            mapToResult(response)
        }.getOrElse { ex ->
            throw BusinessException(ResponseCode.EXTERNAL_API_ERROR, "검색 실패: ${ex.message}")
        }
    }
}
```

> **Kotlin idiom 포인트**:
> - Java 의 `var bool = QueryBuilders.bool(); bool.filter(...); bool.must(...);` → `QueryBuilders.bool().apply { filter { } must { } }`.
> - `c.categoryPath?.let { path -> ... }` — nullable 분기와 capture 동시.
> - `cursor?.let { listOf(FieldValue.of(it.lastSort)) }` — null 일 때 null 전달.
> - `when (c.sort)` exhaustive — enum 의 모든 값 강제 처리.

---

## 7. 자동완성

```kotlin
fun autocomplete(prefix: String, limit: Int): List<String> {
    val response = es.search({ s ->
        s.index("products_v1")
            .query { q -> q.match { m -> m.field("name.auto").query(prefix) } }
            .size(limit)
            .source { src -> src.filter { f -> f.includes("name") } }
    }, ProductDoc::class.java)

    return response.hits().hits().mapNotNull { it.source()?.name }.distinct()
}
```

---

## 9. 함정 모음

(원본 [[../../java-spring/api-design/elasticsearch-integration#9 함정 모음|§9]] 의 10가지 함정 그대로)

### Kotlin 한정 추가 함정

### 함정 11 — Elastic Java client lambda 의 trailing curly
`es.index { it.index(...).id(...).document(doc) }` — Kotlin trailing lambda 가 자연스럽지만, 여러 빌더 메서드 chain 시 `it.x().y().z()` 가 한 줄 안에 들어가야 함. 가독성 위해 `.also { }` 또는 `.apply { }` 보조.

### 함정 12 — `_source` 의 nullable 처리
`response.hits().hits()` 의 `source()` 가 nullable (`ProductDoc?`). `it.source()?.name` 또는 `mapNotNull` 로 안전 처리.

### 함정 13 — Kotlin `when` 의 else 누락
ES sort mode `when` 이 else 없으면 컴파일 에러 (expression). enum 모든 값 처리 또는 `else -> default`.

---

## 10. 운영 체크리스트

- [ ] nori plugin 설치
- [ ] outbox 패턴 + 별도 worker
- [ ] alias 로 index 추상화 (zero-downtime)
- [ ] `search_after` cursor pagination
- [ ] 색인 size / 인덱싱 throughput 모니터
- [ ] 검색 latency p95
- [ ] cluster 최소 3 노드
- [ ] snapshot 정기 (S3)
- [ ] index lifecycle policy
- [ ] Elastic Java client lambda 의 nullable source 처리

---

## 11. 관련

- [[../../java-spring/api-design/elasticsearch-integration|↗ Java 원본]]
- [[webhook-send]] — outbox 패턴 동일
- [[../common/response-envelope]]
- [[api-design|↑ api-design hub]]
