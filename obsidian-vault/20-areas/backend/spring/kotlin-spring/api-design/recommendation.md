---
title: "추천 (넷플릭스·왓챠 스타일) — Kotlin"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T13:20:00+09:00
tags:
  - backend
  - kotlin-spring
  - api-design
  - recipe
  - recommendation
  - personalization
---

# 추천 (넷플릭스·왓챠 스타일) — Kotlin

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + when 분기 + mapNotNull |

**[[api-design|↑ api-design hub]]** · **[[../../java-spring/api-design/recommendation|↗ Java 원본]]**

> 📐 공통: [[../common/response-envelope]]. 본 레시피는 **백엔드 통합 위주** — 모델 학습은 ML 팀.

---

## 1. 추천 시스템 — 백엔드의 역할 / 6. 협업 필터 SQL / 8. 함정

(Java 원본 §1, §6, §8 와 동일)

---

## 2. 이벤트 적재

### 2.1 / 2.2 이벤트 종류 / endpoint

(Java 원본 §2.1, §2.2 와 동일)

### 2.3 Kafka producer 식

```kotlin
@Service
class InteractionEventService(
    private val kafka: KafkaTemplate<String, String>,
    private val json: ObjectMapper,
    private val clock: Clock,
) {
    private val log = LoggerFactory.getLogger(javaClass)

    fun record(
        userId: UserId?,
        type: InteractionType,
        targetType: String,
        targetId: String,
        context: Map<String, Any?>,
    ) {
        val event = InteractionEvent(
            id = UUID.randomUUID().toString(),
            userId = userId?.value,
            type = type, targetType = targetType, targetId = targetId,
            context = context,
            occurredAt = Instant.now(clock),
        )
        runCatching {
            kafka.send(
                "interaction-events",
                userId?.value ?: targetId,                        // partition key
                json.writeValueAsString(event),
            )
        }.onFailure { ex ->
            log.warn("interaction event publish failed", ex)
            // event 적재는 best-effort — 실패해도 비즈니스 흐름 막지 않음
        }
    }
}
```

### 2.4 Controller

```kotlin
@RestController
@RequestMapping("/api/v1/events")
class EventCollectorController(
    private val service: InteractionEventService,
) {

    @PostMapping
    fun record(
        @Valid @RequestBody req: RecordEventRequest,
        @AuthenticationPrincipal user: User?,
    ): ResponseEntity<CommonResponse<Unit>> {
        service.record(user?.id, req.type, req.targetType, req.targetId, req.context.orEmpty())
        return ResponseEntity.accepted().body(
            CommonResponse.success(ResponseCode.OK, "이벤트 적재"),
        )
    }
}

data class RecordEventRequest(
    @field:NotNull val type: InteractionType,
    @field:NotBlank val targetType: String,
    @field:NotBlank val targetId: String,
    val context: Map<String, Any?>? = null,
)
```

---

## 3. 시청기록 (Viewing History) — DB 영속

(Java 원본 §3 의 SQL 그대로)

---

## 4. 추천 결과 조회

```kotlin
@Service
class RecommendationService(
    private val redis: RedisTemplate<String, String>,
    private val products: ProductRepository,
    private val fallback: TrendingFallback,
) {

    fun forUser(userId: UserId, kind: RecKind, limit: Int): List<RecommendedItem> {
        val key = "rec:user:${userId.value}:${kind.name.lowercase()}"
        val typedTuples = redis.opsForZSet()
            .reverseRangeWithScores(key, 0, (limit - 1).toLong())

        if (typedTuples.isNullOrEmpty()) {
            // cold start — fallback (인기 상품 / 신상)
            return fallback.fallback(userId, kind, limit)
        }

        val ids = typedTuples.mapNotNull { it.value?.let(::ProductId) }
        // 메타 hydrate — 한 쿼리
        val byId = products.findAllByIds(ids).associateBy { it.id.value }

        return typedTuples.mapNotNull { tuple ->
            val id = tuple.value ?: return@mapNotNull null
            val p = byId[id] ?: return@mapNotNull null              // 삭제된 상품
            RecommendedItem(p, tuple.score ?: 0.0)
        }
    }
}

enum class RecKind { HOME, SIMILAR, BECAUSE_YOU_VIEWED }
```

> **Kotlin idiom 포인트**:
> - `typedTuples.isNullOrEmpty()` — Java `if (x == null || x.isEmpty())` 한 함수.
> - `mapNotNull { }` — `stream().map().filter(Objects::nonNull).toList()` 한 줄.
> - `findAllByIds(ids).associateBy { it.id.value }` — Java stream + toMap.

### 4.3 추천 API

```kotlin
@GetMapping("/api/v1/recommendations/for-you")
@PreAuthorize("isAuthenticated()")
fun forYou(
    @RequestParam(defaultValue = "HOME") kind: RecKind,
    @RequestParam(defaultValue = "20") @Min(1) @Max(50) limit: Int,
    auth: Authentication,
): ResponseEntity<CommonResponse<List<RecommendedItem>>> =
    CommonResponse.success(
        ResponseCode.OK,
        rec.forUser(UserId(auth.name), kind, limit),
        "추천",
    ).toResponseEntity()
```

---

## 5. Cold Start — 추천 결과 없을 때

```kotlin
@Service
class TrendingFallback(
    private val stats: ProductStatsRepository,
    private val views: UserViewRepository,
) {

    fun fallback(userId: UserId, kind: RecKind, limit: Int): List<RecommendedItem> = when (kind) {
        RecKind.HOME -> {
            // 1순위: 최근 본 카테고리의 인기 상품
            val recentCategories = views.recentCategoriesFor(userId, 5)
            if (recentCategories.isNotEmpty()) {
                stats.topByCategories(recentCategories, limit)
            } else {
                // 2순위: 전체 인기
                stats.topProducts30d(limit)
            }
        }
        RecKind.SIMILAR, RecKind.BECAUSE_YOU_VIEWED -> emptyList()
    }
}
```

> **Kotlin idiom**: `when` expression — Java switch + yield 의 한 줄 표현.

---

## 7. Spring Batch — 야간 갱신 job

```kotlin
@Configuration
class RecomputeRecommendationsJob(
    private val redis: RedisTemplate<String, String>,
    private val views: UserViewRepository,
    private val cooc: ProductCoOccurrenceRepository,
) {
    @Scheduled(cron = "0 0 4 * * *")   // 매일 새벽 4시
    @SchedulerLock(name = "recomputeRec", lockAtMostFor = "PT3H")
    fun run() {
        views.activeUserIdsInLastDays(7, 10_000).forEach { uid ->
            val topItems = computeFor(uid)
            val key = "rec:user:$uid:home"
            redis.delete(key)
            topItems.forEach { item ->
                redis.opsForZSet().add(key, item.id, item.score)
            }
            redis.expire(key, Duration.ofHours(25))   // 다음 갱신까지
        }
    }

    private fun computeFor(userId: String): List<ScoredItem> = TODO("ML 영역")
}
```

---

## 9. 함정 모음

(원본 [[../../java-spring/api-design/recommendation#8 함정 모음|§8]] 의 10가지 함정 그대로)

### Kotlin 한정 추가 함정

### 함정 11 — `typedTuples` 내부 nullable
Spring Data Redis 의 `ZSetOperations.TypedTuple` 의 `value()` / `score()` 둘 다 nullable (`String?` / `Double?`). `?:` fallback 필수.

### 함정 12 — `mapNotNull` 의 `return@mapNotNull null`
중첩 `mapNotNull { tuple -> val x = ... ?: return@mapNotNull null; ... }` — label 명시.

### 함정 13 — `RecKind` enum 변경 시 `when` 자동 안전
`when (kind)` 가 exhaustive 라 enum value 추가하면 컴파일 에러 → fallback 누락 방지.

---

## 10. 운영 체크리스트

(Java 원본 §9 그대로)

---

## 11. 관련

- [[../../java-spring/api-design/recommendation|↗ Java 원본]]
- [[feed-timeline]] — feed 도 추천 결합
- [[cache-redis]] — 추천 결과 캐싱
- [[api-design|↑ api-design hub]]
