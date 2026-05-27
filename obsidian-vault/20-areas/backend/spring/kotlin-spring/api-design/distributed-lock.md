---
title: "분산 락 (Redisson) — Kotlin Spring Boot"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T11:45:00+09:00
tags:
  - backend
  - kotlin-spring
  - api-design
  - recipe
  - distributed-lock
  - redisson
---

# 분산 락 (Redisson) — Kotlin Spring Boot

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + Kotlin extension `withLock` + 함정 + ShedLock |

**[[api-design|↑ api-design hub]]** · **[[../../java-spring/api-design/distributed-lock|↗ Java 원본]]**

> 📐 공통: [[../common/response-envelope]]. 관련 깊이: [[cache-redis#6 Cache Stampede 방지]].

---

## 1. 분산 락 vs JVM 락 vs DB 락

(Java 원본 [[../../java-spring/api-design/distributed-lock#1|§1]] 와 동일 — 인프라 선택 결정은 언어와 무관)

본 레시피는 **Redisson** (가장 널리 쓰임).

---

## 2. 사용 시나리오

| 시나리오 | Lock key | TTL |
| --- | --- | --- |
| 한정수량 재고 차감 | `lock:option:{optionId}` | 3s |
| 사용자 결제 동시 시도 | `lock:user-payment:{userId}` | 10s |
| `@Scheduled` job 다중 인스턴스 | `lock:job:{name}` | job 예상 시간 + 1.5× |
| 외부 API 한 번에 1 호출 | `lock:external-api:{endpoint}` | 5s |
| 캐시 갱신 (stampede 방지) | `lock:cache-refresh:{key}` | 1~3s |
| 쿠폰 first-come-first-serve | `lock:coupon:{couponId}` | 5s |

---

## 3. Redisson 의존성

```kotlin
implementation("org.redisson:redisson-spring-boot-starter:3.27.2")
```

```yaml
spring:
  redis:
    host: ${REDIS_HOST:localhost}
    port: 6379
    timeout: 3000
```

자동으로 `RedissonClient` Bean 생성됨.

---

## 4. 기본 사용 — `tryLock(wait, lease, unit)` + extension function

매번 `try-finally` + `isHeldByCurrentThread()` 체크가 반복되니, Kotlin extension 으로 묶음:

```kotlin
// common/util/RedissonExt.kt
inline fun <T> RLock.withLock(
    waitTime: Long,
    leaseTime: Long,
    unit: TimeUnit,
    onFail: () -> Nothing,
    block: () -> T,
): T {
    val acquired = try {
        tryLock(waitTime, leaseTime, unit)
    } catch (e: InterruptedException) {
        Thread.currentThread().interrupt()
        throw BusinessException(ResponseCode.INTERNAL_SERVER_ERROR, "락 대기 중 인터럽트")
    }
    if (!acquired) onFail()
    try {
        return block()
    } finally {
        if (isHeldByCurrentThread) unlock()
    }
}
```

```kotlin
@Service
class StockService(
    private val redisson: RedissonClient,
    private val options: ProductOptionRepository,
    private val tx: TransactionTemplate,
) {

    fun decreaseStock(optionId: ProductOptionId, qty: Int) {
        val lock = redisson.getLock("lock:option:${optionId.value}")

        lock.withLock(
            waitTime = 3, leaseTime = 5, unit = TimeUnit.SECONDS,
            onFail = {
                throw BusinessException(
                    ResponseCode.RATE_LIMIT_EXCEEDED,
                    "재고 차감 락 획득 실패 — 잠시 후 다시 시도",
                )
            },
        ) {
            tx.executeWithoutResult {
                val opt = options.findById(optionId)
                    ?: throw BusinessException(ResponseCode.NOT_FOUND)
                opt.decreaseStock(qty)
                options.save(opt)
            }
        }
    }
}
```

> **Kotlin idiom 포인트**:
> - `inline fun ... block: () -> T` — Java 의 try-finally + interrupt 처리 + `isHeldByCurrentThread` 체크가 한 곳에 캡슐화.
> - named arguments (`waitTime = 3, leaseTime = 5`) — 매개변수 순서 헷갈림 방지.
> - `onFail: () -> Nothing` — fail path 가 throw 임을 type system 으로 강제.

---

## 5. AOP 식 — `@DistributedLock` annotation

extension 으로 충분한 경우가 많지만, 도메인 코드를 더 깨끗하게 두고 싶으면 annotation + aspect.

```kotlin
// presentation/annotation/DistributedLock.kt
@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class DistributedLock(
    /** SpEL — e.g. "'option:' + #optionId.value" */
    val key: String,
    val waitSeconds: Long = 3,
    val leaseSeconds: Long = 5,
    val throwOnFail: Boolean = true,
)
```

```kotlin
// presentation/aspect/DistributedLockAspect.kt
@Aspect
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)        // 락 aspect 가 @Transactional 보다 outer
class DistributedLockAspect(
    private val redisson: RedissonClient,
) {
    private val log = LoggerFactory.getLogger(javaClass)
    private val parser = SpelExpressionParser()
    private val paramDiscoverer: ParameterNameDiscoverer = DefaultParameterNameDiscoverer()

    @Around("@annotation(distLock)")
    fun around(pjp: ProceedingJoinPoint, distLock: DistributedLock): Any? {
        val sig = pjp.signature as MethodSignature
        val key = "lock:" + resolveKey(distLock.key, sig.method, pjp.args)
        val rlock = redisson.getLock(key)

        return rlock.withLock(
            waitTime = distLock.waitSeconds,
            leaseTime = distLock.leaseSeconds,
            unit = TimeUnit.SECONDS,
            onFail = {
                log.warn("distributed lock failed to acquire: {}", key)
                if (distLock.throwOnFail) {
                    throw BusinessException(ResponseCode.RATE_LIMIT_EXCEEDED, "락 획득 실패: $key")
                }
                // throwOnFail = false 면 null 반환을 흉내내려고 throw Skip... 사실은
                // extension 의 onFail 이 Nothing 이라 이 경로는 throw 되어야 함.
                // 호환 위해 별도 NoLockException 으로 silent skip 가능
                throw NoLockSilentSkip(key)
            },
        ) {
            pjp.proceed()
        }
    }

    private fun resolveKey(spel: String, method: Method, args: Array<Any?>): String {
        val paramNames = paramDiscoverer.getParameterNames(method) ?: return spel
        val ctx = StandardEvaluationContext().apply {
            paramNames.forEachIndexed { i, name -> setVariable(name, args[i]) }
        }
        return parser.parseExpression(spel).getValue(ctx, String::class.java)
            ?: error("SpEL resolved to null: $spel")
    }
}
```

사용:

```kotlin
@Service
class StockService(/* ... */) {

    @DistributedLock(key = "'option:' + #optionId.value", waitSeconds = 3, leaseSeconds = 5)
    @Transactional
    fun decreaseStock(optionId: ProductOptionId, qty: Int) {
        val opt = optionRepo.findById(optionId) ?: throw BusinessException(ResponseCode.NOT_FOUND)
        opt.decreaseStock(qty)
        optionRepo.save(opt)
    }
}
```

> **함정 (트랜잭션 + 락 순서)**: `@DistributedLock` 안에 `@Transactional` 이 있으면 — **락 획득 → 트랜잭션 시작 → DB → 커밋 → 락 해제** 순서. AOP order 가 락 > 트랜잭션 이 되도록 `@Order(Ordered.HIGHEST_PRECEDENCE)` (락 aspect 가 더 outer).

---

## 6. Fencing Token — Redlock 의 한계 해결

Martin Kleppmann 의 비판: TTL 만료 후 다른 노드가 락 획득 → 옛 노드가 작업 마치고 commit → 두 개 동시.

**해결 — fencing token**: 락 획득마다 단조 증가하는 token. 자원 변경 시 token 도 함께 저장 → 작아진 token 거절.

```kotlin
@Service
class FencedStockService(
    private val redisson: RedissonClient,
    private val optionRepo: ProductOptionRepository,
) {

    fun decreaseStock(optionId: ProductOptionId, qty: Int) {
        val lock = redisson.getLock("lock:option:${optionId.value}")
        lock.withLock(
            waitTime = 3, leaseTime = 5, unit = TimeUnit.SECONDS,
            onFail = { throw BusinessException(ResponseCode.RATE_LIMIT_EXCEEDED) },
        ) {
            // Redisson 은 fencing token 직접 제공 X
            // → 별도 AtomicLong (Redis INCR) 으로 token 생성
            val token = redisson.getAtomicLong("token:option:${optionId.value}").incrementAndGet()

            // DB 작업 시 token 도 함께 갱신
            val opt = optionRepo.findById(optionId) ?: throw BusinessException(ResponseCode.NOT_FOUND)
            require(opt.lastFenceToken < token) { "stale fence token" }
            opt.decreaseStockAndUpdateFence(qty, token)
            optionRepo.save(opt)
        }
    }
}
```

→ TTL 만료 시나리오에서도 옛 노드의 commit 이 거절됨.

> 실전에선 대부분 fencing token 까지는 불필요 (TTL 충분히 길게 + 트랜잭션 짧게 = 99% OK). 절대 안전성 필요 시 도입.

---

## 7. TTL 의 함정 — "락 hold time < TTL"

```
TTL = 5s, 작업 = 4s          → OK
TTL = 5s, 작업 = 6s 지연      → TTL 만료, 다른 노드가 락 획득, 옛 노드 commit = 동시
```

**규칙**:
1. **TTL = 작업 예상 시간 + 안전 마진 (3~5x)**
2. **외부 IO 락 안에 두지 말 것** — 예측 불가
3. 트랜잭션 짧게, lock 짧게

```kotlin
@DistributedLock(key = "...", leaseSeconds = 5)
fun decreaseStock(/* ... */) {
    // ❌ 락 안에 외부 API 호출 X
    // pgClient.confirm(...)            // 5초 timeout → TTL 위험

    // ✅ DB 만
    opt.decreaseStock(qty)
    optionRepo.save(opt)
}
```

---

## 8. `@Scheduled` 분산 — ShedLock

```kotlin
implementation("net.javacrumbs.shedlock:shedlock-spring:5.13.0")
implementation("net.javacrumbs.shedlock:shedlock-provider-redis-spring:5.13.0")
```

```kotlin
@Configuration
@EnableScheduling
@EnableSchedulerLock(defaultLockAtMostFor = "PT10M")
class SchedulerConfig {
    @Bean
    fun lockProvider(cf: RedisConnectionFactory): LockProvider = RedisLockProvider(cf)
}

@Component
class ExpireOrdersJob {
    @Scheduled(fixedDelay = 60_000)
    @SchedulerLock(name = "expireOrders", lockAtLeastFor = "30s", lockAtMostFor = "5m")
    fun run() {
        // ...
    }
}
```

→ 인스턴스 N개 중 한 곳만 실행. 다른 곳은 같은 분에 무시.

본 레시피의 `@DistributedLock` 은 **요청 흐름의 critical section** 용. 스케줄 job 은 ShedLock.

---

## 9. 테스트

```kotlin
@Test
fun `concurrent_stock_decrease_does_not_oversell`() {
    val option = optionFixture.createWithStock(50)
    val users = userFixture.createMany(100)

    val executor = Executors.newFixedThreadPool(20)
    val successes = AtomicInteger()
    val latch = CountDownLatch(100)

    users.forEach { _ ->
        executor.submit {
            runCatching { stockService.decreaseStock(option.id, 1) }
                .onSuccess { successes.incrementAndGet() }
            latch.countDown()
        }
    }
    latch.await(30, TimeUnit.SECONDS)
    executor.shutdown()

    assertThat(successes.get()).isEqualTo(50)
    assertThat(optionFixture.reload(option.id).stock).isEqualTo(0)
}
```

> **Kotlin idiom**: `runCatching { } .onSuccess { }` — try/catch + count 한 chain.

---

## 10. 함정 모음

(원본 [[../../java-spring/api-design/distributed-lock#10 함정 모음|§10]] 의 10가지 함정 그대로 적용 — Redisson / TTL / fencing 이슈는 언어와 무관)

### Kotlin 한정 추가 함정

### 함정 11 — `inline fun withLock` 안에서 `return` 사용
`inline` 함수의 `block: () -> T` 안에서 `return` 을 쓰면 **non-local return** — 호출한 메서드까지 반환됨. 의도와 다르게 락이 풀린 후 메서드 종료. 의도적이라면 OK, 아니면 `return@withLock` 명시.

### 함정 12 — `@DistributedLock` 메서드가 final / private
`kotlin("plugin.spring")` 적용 안 하면 default `final` 이라 AOP proxy 가 안 됨. plugin 적용 + `internal` / `public` 가시성 유지.

### 함정 13 — Annotation 의 `String` 파라미터에 string template
`@DistributedLock(key = "lock:$id")` — annotation 의 `String` 파라미터는 **컴파일 타임 상수**여야 함. Kotlin string template 의 `$` 는 runtime evaluation 이라 컴파일 에러. **SpEL 만** 사용 (`"'lock:' + #id"`).

---

## 11. 운영 체크리스트

- [ ] `isHeldByCurrentThread` 체크 후 unlock (extension 이 자동 처리)
- [ ] TTL 이 작업 예상 시간 × 3~5
- [ ] 락 안에 외부 IO X
- [ ] AOP order — `@DistributedLock` > `@Transactional`
- [ ] Redis 모니터링 — 락 wait 시간 / 충돌율
- [ ] Redis Sentinel / Cluster 구성 (단일 SPOF X)
- [ ] 정기 락 분포 분석 (어느 키 가장 자주 충돌)
- [ ] `@Scheduled` 는 ShedLock 사용
- [ ] fencing token — 절대 안전성 필요한 도메인만
- [ ] `inline fun withLock` 의 non-local return 주의

---

## 12. 관련

- [[cache-redis#6 Cache Stampede 방지]] — 락 활용
- [[../../java-spring/api-design/distributed-lock|↗ Java 원본]]
- [[api-design|↑ api-design hub]]
