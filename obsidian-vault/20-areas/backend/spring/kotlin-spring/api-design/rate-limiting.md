---
title: "Rate Limiting (Bucket4j + Redis) — Kotlin"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T13:00:00+09:00
tags:
  - backend
  - kotlin-spring
  - api-design
  - recipe
  - rate-limit
  - bucket4j
---

# Rate Limiting (Bucket4j + Redis) — Kotlin

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + sealed Dimension + when 분기 |

**[[api-design|↑ api-design hub]]** · **[[../../java-spring/api-design/rate-limiting|↗ Java 원본]]**

> 📐 공통: [[../common/response-envelope]] · [[../common/security-config]].

---

## 1. 무엇을 만드는가 / 1.1 보호 대상 / 1.2 알고리즘 / 6. 응답 형식 / 8. 함정

(Java 원본 §1, §6, §8 와 동일)

---

## 2. 의존성

```kotlin
implementation("com.bucket4j:bucket4j-redis:8.10.1")
implementation("com.bucket4j:bucket4j-core:8.10.1")
implementation("org.springframework.boot:spring-boot-starter-data-redis")
```

---

## 3. Rate Limit Service (port + adapter)

```kotlin
// domain/ratelimit/RateLimiter.kt
interface RateLimiter {
    /** 토큰 1개 소비 시도. allowed=true 성공, false 거절. */
    fun tryConsume(key: String, policy: RateLimitPolicy): ConsumptionResult

    data class ConsumptionResult(
        val allowed: Boolean,
        val remainingTokens: Long,
        val retryAfter: Duration?,
    )
}

data class RateLimitPolicy(val capacity: Long, val refillPeriod: Duration) {
    companion object {
        fun of(capacity: Long, refill: Duration): RateLimitPolicy =
            RateLimitPolicy(capacity, refill)
    }
}
```

### 3.1 Bucket4j + Redis adapter

```kotlin
@Component
class Bucket4jRedisRateLimiter(
    private val redisson: RedissonClient,
) : RateLimiter {

    override fun tryConsume(key: String, policy: RateLimitPolicy): RateLimiter.ConsumptionResult {
        val proxyManager = RedissonBasedProxyManager.builderFor(redisson)
            .withExpirationStrategy(
                ExpirationAfterWriteStrategy.basedOnTimeForRefillingBucketUpToMax(Duration.ofHours(1)),
            )
            .build()

        val config = BucketConfiguration.builder()
            .addLimit { limit ->
                limit.capacity(policy.capacity).refillIntervally(policy.capacity, policy.refillPeriod)
            }
            .build()

        val bucket = proxyManager.builder().build(key) { config }
        val probe = bucket.tryConsumeAndReturnRemaining(1)

        return if (probe.isConsumed) {
            RateLimiter.ConsumptionResult(true, probe.remainingTokens, null)
        } else {
            RateLimiter.ConsumptionResult(false, 0, Duration.ofNanos(probe.nanosToWaitForRefill))
        }
    }
}
```

> **Kotlin idiom**: Java fluent builder lambda `.addLimit(limit -> limit.x())` → `.addLimit { it.x() }` trailing.

---

## 4. 적용 — Filter (글로벌)

```kotlin
@Component
class RateLimitFilter(
    private val limiter: RateLimiter,
    private val objectMapper: ObjectMapper,
) : OncePerRequestFilter() {

    enum class Dimension { IP, USER, IP_PLUS_USER }
    data class RateLimitRule(val pattern: String, val dimension: Dimension, val policy: RateLimitPolicy)

    companion object {
        private val PATH = AntPathMatcher()

        private val RULES = listOf(
            RateLimitRule("/api/v1/auth/login",                     Dimension.IP,   RateLimitPolicy.of(5,  Duration.ofMinutes(1))),
            RateLimitRule("/api/v1/auth/signup",                    Dimension.IP,   RateLimitPolicy.of(3,  Duration.ofHours(1))),
            RateLimitRule("/api/v1/auth/password-reset/request",    Dimension.IP,   RateLimitPolicy.of(3,  Duration.ofHours(1))),
            RateLimitRule("/api/v1/auth/verify/email/request",      Dimension.USER, RateLimitPolicy.of(3,  Duration.ofHours(1))),
            RateLimitRule("/api/v1/payments/confirm",               Dimension.USER, RateLimitPolicy.of(10, Duration.ofMinutes(1))),
        )
    }

    override fun doFilterInternal(req: HttpServletRequest, res: HttpServletResponse, chain: FilterChain) {
        val matched = RULES.firstOrNull { PATH.match(it.pattern, req.servletPath) }
        if (matched != null) {
            val key = buildKey(req, matched.dimension, matched.pattern)
            val result = limiter.tryConsume(key, matched.policy)
            if (!result.allowed) {
                writeRateLimitResponse(res, result.retryAfter ?: Duration.ofSeconds(1))
                return
            }
        }
        chain.doFilter(req, res)
    }

    private fun buildKey(req: HttpServletRequest, dim: Dimension, pattern: String): String {
        val prefix = "rl:$pattern:"
        return when (dim) {
            Dimension.IP -> "${prefix}ip:${req.clientIp()}"
            Dimension.USER -> "${prefix}u:${currentUserId() ?: "anonymous"}"
            Dimension.IP_PLUS_USER ->
                "${prefix}ip:${req.clientIp()}:u:${currentUserId() ?: "anonymous"}"
        }
    }

    private fun writeRateLimitResponse(res: HttpServletResponse, retryAfter: Duration) {
        val body = CommonResponse.fail<Unit>(
            ResponseCode.RATE_LIMIT_EXCEEDED,
            "요청이 너무 많습니다. 잠시 후 다시 시도해 주세요.",
        )
        res.status = HttpServletResponse.SC_TOO_MANY_REQUESTS
        res.contentType = MediaType.APPLICATION_JSON_VALUE
        res.characterEncoding = StandardCharsets.UTF_8.name()
        res.setHeader("Retry-After", maxOf(1, retryAfter.toSeconds()).toString())
        objectMapper.writeValue(res.outputStream, body)
    }
}

private fun HttpServletRequest.clientIp(): String =
    getHeader("X-Forwarded-For")
        ?.takeIf { it.isNotBlank() }
        ?.substringBefore(",")
        ?.trim()
        ?: remoteAddr

private fun currentUserId(): String? =
    SecurityContextHolder.getContext().authentication?.takeIf { it.isAuthenticated }?.name
```

> **Kotlin idiom 포인트**:
> - `firstOrNull { }` — Stream `findFirst()` 보다 짧음.
> - `when (dim)` — exhaustive 검사가 enum 의 모든 case 강제.
> - extension `HttpServletRequest.clientIp()` — 헬퍼를 receiver.

`SecurityConfig` 에 등록:
```kotlin
http.addFilterBefore<JwtAuthenticationFilter>(rateLimitFilter)
```

→ JWT 검증 전에 rate limit (DB 조회 비용 절약).

### 4.2 Annotation — `@RateLimit` (메서드 단)

```kotlin
@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class RateLimit(
    val capacity: Int,
    val periodSeconds: Int,
    val dimension: Dimension = Dimension.USER,
    val keySuffix: String = "",        // SpEL 식 ("#email") 도 가능
)

@Aspect
@Component
class RateLimitAspect(
    private val limiter: RateLimiter,
) {
    private val parser = SpelExpressionParser()

    @Around("@annotation(rateLimit)")
    fun around(pjp: ProceedingJoinPoint, rateLimit: RateLimit): Any? {
        val key = buildKey(pjp, rateLimit)
        val policy = RateLimitPolicy.of(
            rateLimit.capacity.toLong(),
            Duration.ofSeconds(rateLimit.periodSeconds.toLong()),
        )
        val result = limiter.tryConsume(key, policy)
        if (!result.allowed) {
            throw BusinessException(ResponseCode.RATE_LIMIT_EXCEEDED, "요청이 너무 많습니다.")
        }
        return pjp.proceed()
    }

    private fun buildKey(pjp: ProceedingJoinPoint, anno: RateLimit): String {
        // pjp.args 와 SpEL parser 로 keySuffix 평가 — Java 원본 §4.2 참고
        TODO("SpEL 평가 + dimension 별 key")
    }
}
```

사용:
```kotlin
@PostMapping("/login")
@RateLimit(capacity = 5, periodSeconds = 60, dimension = Dimension.IP_PLUS_USER, keySuffix = "#req.email")
fun login(@RequestBody req: LoginRequest): ResponseEntity<*> { /* ... */ }
```

### 4.3 Gateway / Nginx 단

(Java 원본 [[../../java-spring/api-design/rate-limiting#4.3 Gateway / Nginx 단|§4.3]] 동일)

---

## 5. SecurityConfig 등록 / 7. user 역할 별 정책

(Java 원본 §5, §7 와 동일 — Kotlin DSL 의 `addFilterBefore<JwtAuthenticationFilter>(...)` 만 다름)

---

## 9. 함정 모음

(원본 [[../../java-spring/api-design/rate-limiting#8 함정 모음|§8]] 의 10가지 함정 그대로)

### Kotlin 한정 추가 함정

### 함정 11 — Bucket4j lambda 의 `it` chain
`.addLimit { it.capacity(...).refillIntervally(...) }` — `it` 가 한 expression 안에 다 들어가야 함. 줄 길이 신경.

### 함정 12 — Annotation parameter 의 Int vs Long
Java annotation 은 `int` / `long` 구분 — Kotlin 에선 `Int` / `Long`. `RateLimit.capacity: Int` 를 `RateLimitPolicy.capacity: Long` 로 변환 시 `.toLong()` 필수.

### 함정 13 — `firstOrNull { }` 의 short-circuit
`RULES.filter { ... }.firstOrNull()` 는 전체 RULES 평가, `firstOrNull { ... }` 는 조기 종료. 후자 권장.

---

## 10. 운영 체크리스트

(Java 원본 §9 그대로)

---

## 11. 관련

- [[../../java-spring/api-design/rate-limiting|↗ Java 원본]]
- [[distributed-lock]] — 비슷한 Redis 기반 패턴
- [[../common/security-config]]
- [[api-design|↑ api-design hub]]
