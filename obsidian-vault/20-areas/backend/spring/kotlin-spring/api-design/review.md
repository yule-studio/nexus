---
title: "리뷰 / 평점 (무신사·쿠팡 스타일) — Kotlin"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T13:30:00+09:00
tags:
  - backend
  - kotlin-spring
  - api-design
  - recipe
  - review
  - rating
---

# 리뷰 / 평점 (무신사·쿠팡 스타일) — Kotlin

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + sealed event + 함정 |

**[[api-design|↑ api-design hub]]** · **[[../../java-spring/api-design/review|↗ Java 원본]]**

> 📐 공통: [[../common/response-envelope]] · [[../common/security-config]].

---

## 1. 무엇을 만드는가 / 2. 도메인·DB / 4. 정렬·필터 SQL / 6. Moderation Admin / 7. 함정

(Java 원본 §1, §2, §4 SQL, §6, §7 와 동일)

---

## 3. UseCase

### 3.1 CreateReviewUseCase

```kotlin
@Service
class CreateReviewUseCase(
    private val reviews: ReviewRepository,
    private val orderItems: OrderItemRepository,
    private val ids: IdGenerator,
    private val clock: Clock,
    private val events: ApplicationEventPublisher,
) {

    @Transactional
    fun handle(
        userId: UserId, productId: ProductId, orderItemId: OrderItemId,
        rating: Int, content: String, imageUrls: List<String>,
    ): ReviewId {
        // 1. 구매 확인
        val orderItem = orderItems.findById(orderItemId)
            ?: throw BusinessException(ResponseCode.NOT_FOUND, "주문 항목")
        if (!orderItem.belongsToBuyer(userId))
            throw BusinessException(ResponseCode.FORBIDDEN, "본인 주문만 리뷰 가능")
        if (orderItem.productId != productId)
            throw BusinessException(ResponseCode.BAD_REQUEST, "주문한 상품이 아닙니다.")
        if (!orderItem.isReviewable())
            throw BusinessException(ResponseCode.BAD_REQUEST, "리뷰 가능 상태가 아닙니다 (배송 완료 후).")

        // 2. 1인1리뷰 — DB UNIQUE 가 진실
        if (reviews.existsByUserAndProduct(userId, productId))
            throw BusinessException(ResponseCode.DUPLICATE_DATA, "이미 리뷰를 작성했습니다.")

        require(rating in 1..5) { "평점은 1~5" }
        require(content.isNotBlank() && content.length <= 2000) { "리뷰 내용 1~2000자" }
        require(imageUrls.size <= 5) { "이미지 최대 5장" }

        val review = Review.create(
            id = ReviewId(ids.next()),
            userId = userId, productId = productId, orderItemId = orderItemId,
            rating = rating, content = content,
            imageUrls = imageUrls,
            now = Instant.now(clock),
        )

        runCatching {
            reviews.save(review)
        }.onFailure { ex ->
            if (ex is DataIntegrityViolationException) {
                throw BusinessException(ResponseCode.DUPLICATE_DATA, "이미 리뷰를 작성했습니다.")
            }
            throw ex
        }

        // 3. 평점 집계는 이벤트로 분리
        events.publishEvent(ReviewCreated(review.id, productId, rating, Instant.now(clock)))

        return review.id
    }
}

sealed interface ReviewEvent : DomainEvent
data class ReviewCreated(
    val reviewId: ReviewId, val productId: ProductId, val rating: Int, val occurredAt: Instant,
) : ReviewEvent

data class ReviewDeleted(
    val reviewId: ReviewId, val productId: ProductId, val occurredAt: Instant,
) : ReviewEvent
```

> **Kotlin idiom 포인트**:
> - `sealed interface ReviewEvent` — `when (event)` 가 모든 sub-type 처리 강제.
> - `require(rating in 1..5)` — range 검증을 IntRange 로 짧게.

### 3.2 평점 집계 listener

```kotlin
@Component
class ProductStatsRecalculator(
    private val stats: ProductStatsRepository,
    private val reviews: ReviewRepository,
) {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Async
    fun on(event: ReviewEvent) {
        val productId = when (event) {
            is ReviewCreated -> event.productId
            is ReviewDeleted -> event.productId
        }
        val aggregate = reviews.aggregateForProduct(productId)
        stats.upsert(productId, aggregate.count, aggregate.average)
    }
}
```

> **Kotlin idiom**: `when (event)` 이 `ReviewEvent` sealed 의 모든 sub-type 강제 — 새 event 추가 시 컴파일 에러로 누락 방지.

> **함정**: 인기 상품에선 전체 재계산이 O(N). 증분 update 또는 1분 batch.

### 3.3 ToggleHelpfulUseCase

```kotlin
@Service
class ToggleHelpfulUseCase(
    private val helpfuls: ReviewHelpfulRepository,
    private val reviews: ReviewRepository,
    private val clock: Clock,
) {

    @Transactional
    fun handle(reviewId: ReviewId, userId: UserId): Boolean {
        val review = reviews.findById(reviewId)
            ?: throw BusinessException(ResponseCode.NOT_FOUND, "리뷰")
        if (review.isSelfWritten(userId))
            throw BusinessException(ResponseCode.BAD_REQUEST, "자기 리뷰에 도움돼요 불가")

        val existed = helpfuls.exists(reviewId, userId)
        if (existed) {
            helpfuls.delete(reviewId, userId)
            review.decrementHelpful()
        } else {
            helpfuls.save(reviewId, userId, Instant.now(clock))
            review.incrementHelpful()
        }
        reviews.save(review)
        return !existed
    }
}
```

### 3.4 ReportReviewUseCase

```kotlin
@Service
class ReportReviewUseCase(
    private val reports: ReviewReportRepository,
    private val reviews: ReviewRepository,
    private val ids: IdGenerator,
    private val clock: Clock,
) {
    companion object {
        private const val AUTO_HIDE_THRESHOLD = 5
    }

    @Transactional
    fun handle(reviewId: ReviewId, reporterId: UserId, reason: String, detail: String?) {
        val review = reviews.findById(reviewId)
            ?: throw BusinessException(ResponseCode.NOT_FOUND, "리뷰")

        runCatching {
            reports.save(
                ReviewReport(
                    id = ReviewReportId(ids.next()),
                    reviewId = reviewId,
                    reporterId = reporterId,
                    reason = reason, detail = detail,
                    status = "PENDING",
                    createdAt = Instant.now(clock),
                ),
            )
        }.onFailure { ex ->
            if (ex is DataIntegrityViolationException)
                throw BusinessException(ResponseCode.DUPLICATE_DATA, "이미 신고했습니다.")
            throw ex
        }

        val newCount = review.incrementReportCount()
        if (newCount >= AUTO_HIDE_THRESHOLD) review.hideByReport()
        reviews.save(review)
    }
}
```

---

## 5. Controller

```kotlin
@Tag(name = "리뷰")
@RestController
@RequestMapping("/api/v1")
class ReviewController(
    private val create: CreateReviewUseCase,
    private val query: ReviewQueryService,
    private val toggleHelpful: ToggleHelpfulUseCase,
    private val report: ReportReviewUseCase,
) {

    @PostMapping("/products/{productId}/reviews")
    @PreAuthorize("isAuthenticated()")
    fun create(
        @PathVariable productId: String,
        @Valid @RequestBody req: CreateReviewRequest,
        auth: Authentication,
    ): ResponseEntity<CommonResponse<Map<String, String>>> {
        val id = create.handle(
            UserId(auth.name),
            ProductId(productId), OrderItemId(req.orderItemId),
            req.rating, req.content, req.imageUrls,
        )
        return CommonResponse.success(
            ResponseCode.OK,
            mapOf("reviewId" to id.value),
            "리뷰 작성 완료",
        ).toResponseEntity()
    }

    @PostMapping("/reviews/{reviewId}/helpful")
    @PreAuthorize("isAuthenticated()")
    fun helpful(
        @PathVariable reviewId: String,
        auth: Authentication,
    ): ResponseEntity<CommonResponse<Map<String, Boolean>>> {
        val active = toggleHelpful.handle(ReviewId(reviewId), UserId(auth.name))
        return CommonResponse.success(
            ResponseCode.OK,
            mapOf("active" to active),
            if (active) "도움돼요 표시" else "취소",
        ).toResponseEntity()
    }

    @PostMapping("/reviews/{reviewId}/report")
    @PreAuthorize("isAuthenticated()")
    fun reportReview(
        @PathVariable reviewId: String,
        @Valid @RequestBody req: ReportRequest,
        auth: Authentication,
    ): ResponseEntity<CommonResponse<Unit>> {
        report.handle(ReviewId(reviewId), UserId(auth.name), req.reason, req.detail)
        return CommonResponse.success<Unit>(ResponseCode.OK, "신고 접수").toResponseEntity()
    }
}

data class CreateReviewRequest(
    @field:NotBlank val orderItemId: String,
    @field:Min(1) @field:Max(5) val rating: Int,
    @field:NotBlank @field:Size(min = 1, max = 2000) val content: String,
    @field:Size(max = 5) val imageUrls: List<@Pattern(regexp = "^https://.*") String> = emptyList(),
)

data class ReportRequest(
    @field:NotBlank val reason: String,
    @field:Size(max = 1000) val detail: String? = null,
)
```

---

## 8. 함정 모음

(원본 [[../../java-spring/api-design/review#7 함정 모음|§7]] 의 10가지 함정 그대로)

### Kotlin 한정 추가 함정

### 함정 11 — `sealed interface ReviewEvent` 의 sub-type 분기 누락
`when (event) { is ReviewCreated -> ... }` 만 있고 `is ReviewDeleted` 없으면 컴파일 에러 (exhaustive). 새 event 추가 시 자동 안전.

### 함정 12 — `require(rating in 1..5)`
`IllegalArgumentException` 으로 throw — `BusinessException` 으로 wrap 필요 시 try/catch 또는 별도 helper.

### 함정 13 — `List<@Pattern(...) String>` 의 validation target
data class parameter 의 generic 안 어노테이션 — `@field:` target 안 됨. `@Pattern` 만으로 `String` 자체에 붙음 (Kotlin compiler 가 자동).

---

## 9. 운영 체크리스트

(Java 원본 §8 그대로)

---

## 10. 관련

- [[../../java-spring/api-design/review|↗ Java 원본]]
- [[file-upload-s3]] — 리뷰 이미지
- [[api-design|↑ api-design hub]]
