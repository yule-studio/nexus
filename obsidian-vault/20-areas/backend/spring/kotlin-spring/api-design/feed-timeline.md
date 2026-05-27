---
title: "피드 / 타임라인 (네이버 카페·인스타 스타일) — Kotlin"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T12:25:00+09:00
tags:
  - backend
  - kotlin-spring
  - api-design
  - recipe
  - feed
  - timeline
---

# 피드 / 타임라인 (네이버 카페·인스타 스타일) — Kotlin

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + Kotlin idiom + 함정 |

**[[api-design|↑ api-design hub]]** · **[[../../java-spring/api-design/feed-timeline|↗ Java 원본]]**

> 📐 공통: [[../common/response-envelope]]. cursor 패턴 동일: [[../../java-spring/api-design/feed-timeline#2 피드 생성 전략|product-search §2]].

---

## 1. 무엇을 만드는가 / 2. 피드 생성 전략 / 3. DB 스키마 / 7. 트렌딩

(Java 원본 §1, §2, §3, §7 와 동일 — API spec / 도메인 모델 / DB / SQL 은 언어와 무관)

---

## 4. Feed 조회 (Fan-out on Read)

```kotlin
interface FeedQueryRepository : JpaRepository<PostJpaEntity, String> {

    @Query(
        value = """
            SELECT p.id, p.author_id AS authorId, p.title, p.content,
                   p.like_count AS likeCount, p.comment_count AS commentCount,
                   p.created_at AS createdAt
            FROM posts p
            WHERE p.status = 'PUBLISHED'
              AND p.author_id IN (
                  SELECT followee_id FROM follows WHERE follower_id = :userId
                  UNION SELECT CAST(:userId AS CHAR(26))
              )
              AND ((:cursorCreated IS NULL)
                   OR (p.created_at, p.id) < (:cursorCreated, :cursorId))
            ORDER BY p.created_at DESC, p.id DESC
            LIMIT :limit
        """,
        nativeQuery = true,
    )
    fun feed(
        @Param("userId") userId: String,
        @Param("cursorCreated") cursorCreated: Instant?,
        @Param("cursorId") cursorId: String?,
        @Param("limit") limit: Int,
    ): List<FeedView>
}
```

> **함정**: `follower 수 × 평균 post 수` 가 큰 사용자는 느림. **§6 Hybrid** 로.

---

## 5. 좋아요 토글 — atomic UPDATE

```kotlin
@Service
class TogglePostLikeUseCase(
    private val postLikes: PostLikeRepository,
    private val postRepo: PostRepository,
    private val clock: Clock,
) {

    @Transactional
    fun handle(postId: PostId, userId: UserId): Boolean {
        val existed = postLikes.exists(postId, userId)
        if (existed) {
            postLikes.delete(postId, userId)
            postRepo.decrementLikeCount(postId)
        } else {
            runCatching {
                postLikes.save(postId, userId, Instant.now(clock))
                postRepo.incrementLikeCount(postId)
            }.onFailure { ex ->
                if (ex !is DataIntegrityViolationException) throw ex
                // 동시 클릭 — 이미 존재
            }
        }
        return !existed
    }
}
```

```kotlin
// JPA modifying — 동시성 안전
interface PostRepository : JpaRepository<PostJpaEntity, String> {

    @Modifying(clearAutomatically = true)
    @Query("update PostJpaEntity p set p.likeCount = p.likeCount + 1 where p.id = :id")
    fun incrementLikeCount(@Param("id") id: String): Int
}
```

> **Kotlin idiom**: `runCatching { } / .onFailure { if (it !is X) throw it }` — 특정 예외만 무시.

---

## 6. Hybrid Fan-out (VIP user)

```kotlin
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
@Async
fun onPostCreated(event: PostCreated) {
    val author = userRepo.findById(event.authorId) ?: return
    if (author.followerCount > 10_000) {
        // Hybrid Push — Kafka 로 비동기
        kafkaProducer.send("feed-fanout", FanoutMessage(event.postId, event.authorId))
    }
    // 일반 user 의 글은 read 시점에 직접 query (push X)
}

// Kafka consumer
@KafkaListener(topics = ["feed-fanout"])
fun consume(msg: FanoutMessage) {
    val followers = followRepo.followerIdsOf(msg.authorId)
    feedCacheRepo.batchInsert(followers, msg.postId, Instant.now())
}
```

Feed 조회 (Java 원본 §6 SQL 그대로):
```sql
SELECT * FROM (
    SELECT post_id, created_at FROM feed_cache WHERE user_id = :userId
    UNION
    SELECT p.id, p.created_at FROM posts p
    WHERE p.author_id IN (
        SELECT followee_id FROM follows WHERE follower_id = :userId
        AND followee_id NOT IN (SELECT id FROM users WHERE is_vip = true)
    )
) combined
ORDER BY created_at DESC LIMIT 20;
```

> 복잡. **시작은 Fan-out on Read** + 성장 시 Hybrid.

---

## 8. Controller

```kotlin
@RestController
@RequestMapping("/api/v1")
class FeedController(
    private val feed: FeedQueryService,
    private val toggleLike: TogglePostLikeUseCase,
) {

    @GetMapping("/feed")
    @PreAuthorize("isAuthenticated()")
    fun myFeed(
        @RequestParam(required = false) cursor: String?,
        @RequestParam(defaultValue = "20") @Min(1) @Max(50) limit: Int,
        auth: Authentication,
    ): ResponseEntity<CommonResponse<FeedResponse>> {
        val response = feed.myFeed(UserId(auth.name), cursor, limit)
        return CommonResponse.success(ResponseCode.OK, response, "조회").toResponseEntity()
    }

    @PostMapping("/posts/{postId}/like")
    @PreAuthorize("isAuthenticated()")
    fun like(
        @PathVariable postId: String,
        auth: Authentication,
    ): ResponseEntity<CommonResponse<Map<String, Boolean>>> {
        val active = toggleLike.handle(PostId(postId), UserId(auth.name))
        return CommonResponse.success(
            ResponseCode.OK,
            mapOf("active" to active),
            if (active) "좋아요" else "취소",
        ).toResponseEntity()
    }
}
```

> **Kotlin idiom**: `if (active) "좋아요" else "취소"` — Java ternary `active ? "좋아요" : "취소"` 와 동등. expression 으로 자연스럽게 args 안.

---

## 9. 함정 모음

(원본 [[../../java-spring/api-design/feed-timeline#9 함정 모음|§9]] 의 10가지 함정 그대로)

### Kotlin 한정 추가 함정

### 함정 11 — `@KafkaListener(topics = ...)` 의 array 표기
Java `@KafkaListener(topics = "x")` 는 single string OK, Kotlin 에선 `@KafkaListener(topics = ["x"])` array literal 필요.

### 함정 12 — JPA Entity 를 data class
post / like 같은 entity 가 data class 면 equals/hashCode 가 모든 필드 → lazy `comments`/`likes` collection 이 로딩됨. 일반 `class` + `open class` ([[../common/security-config#함정 14|함정 14]] 참고).

---

## 10. 운영 체크리스트

- [ ] like_count / comment_count 비정규화
- [ ] atomic UPDATE (race 안전)
- [ ] cursor pagination
- [ ] feed 의 follow JOIN 인덱스
- [ ] 트렌딩 캐시
- [ ] 댓글 1-level nesting
- [ ] VIP 식별 + push 분리 (Hybrid)
- [ ] 좋아요 / 댓글 알림 digest
- [ ] `@KafkaListener` array literal

---

## 11. 관련

- [[../../java-spring/api-design/feed-timeline|↗ Java 원본]]
- [[recommendation]] — 추천 feed 변형
- [[file-upload-s3]] — 글 이미지
- [[api-design|↑ api-design hub]]
