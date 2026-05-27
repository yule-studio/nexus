---
title: "Redis 캐시 (cache-aside + stampede 방지) — Kotlin"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T11:15:00+09:00
tags:
  - backend
  - kotlin-spring
  - api-design
  - recipe
  - cache
  - redis
---

# Redis 캐시 (cache-aside + stampede 방지) — Kotlin

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + Kotlin idiom (apply / runCatching / coroutines optional) |

**[[api-design|↑ api-design hub]]** · **[[../../java-spring/api-design/cache-redis|↗ Java 원본]]**

> 📐 공통: [[../common/response-envelope]]. 락 깊이: [[distributed-lock]].

---

## 1. 캐시 패턴 4가지

| 패턴 | 설명 | 사용처 |
| --- | --- | --- |
| **Cache-aside (Lazy loading)** | 앱이 캐시 확인 → miss 시 DB → 캐시에 적재 | **본 레시피 기본** |
| Read-through | 캐시가 DB 미러링 — 앱은 캐시만 호출 | Redis Module / 인프라성 |
| Write-through | write 가 DB + 캐시 둘 다 | 강한 일관성 필요 |
| Write-behind | write 는 캐시만, 나중에 DB 동기화 | 잦은 update / 임시성 |

→ **Cache-aside** = 가장 흔하고 단순. 본 레시피.

---

## 2. 무엇을 캐시할까

| 후보 | 적합 | 키 / TTL |
| --- | --- | --- |
| **상품 상세** (read 폭주) | ✅ | `product:{id}` / 10m |
| **사용자 profile** (자주 조회) | ✅ | `user:{id}` / 5m |
| **카테고리 트리** (거의 안 변함) | ✅ | `category:tree` / 1h |
| **인기 상품 랭킹** | ✅ | `product:top:{cat}` / 5m |
| **검색 결과** (자주 같은 검색) | ⚠️ | `search:{hash}` / 30s |
| 사용자 카트 | ❌ (Cart 자체가 Redis) | — |
| 결제 / 주문 변경 | ❌ (강한 일관성) | — |

기본 룰:
- **자주 read + 가끔 write** → 캐시
- **자주 write + 자주 read** → 캐시 안 함 또는 짧은 TTL
- **강한 일관성** (돈 / 재고) → 캐시 안 함

---

## 3. 설정 — Spring Data Redis + RedisCacheManager

```kotlin
implementation("org.springframework.boot:spring-boot-starter-data-redis")
implementation("org.springframework.boot:spring-boot-starter-cache")
```

```yaml
spring:
  redis:
    host: ${REDIS_HOST:localhost}
    port: 6379
    timeout: 3000
  cache:
    type: redis
    redis:
      time-to-live: 600000              # 10 min default
      key-prefix: "shop:cache:"
      use-key-prefix: true
```

```kotlin
// common/config/CacheConfig.kt
@Configuration
@EnableCaching
class CacheConfig {

    @Bean
    fun redisCacheManager(connectionFactory: RedisConnectionFactory): RedisCacheManager {
        val defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .disableCachingNullValues()                  // null 안 캐시
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(StringRedisSerializer()),
            )
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(GenericJackson2JsonRedisSerializer(objectMapper())),
            )

        // 캐시별 TTL 차등
        val configs = mapOf(
            "product:detail" to defaultConfig.entryTtl(Duration.ofMinutes(10)),
            "user:profile"   to defaultConfig.entryTtl(Duration.ofMinutes(5)),
            "category:tree"  to defaultConfig.entryTtl(Duration.ofHours(1)),
            "product:top"    to defaultConfig.entryTtl(Duration.ofMinutes(5)),
        )

        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(configs)
            .build()
    }

    private fun objectMapper(): ObjectMapper =
        ObjectMapper()
            .registerModule(JavaTimeModule())
            .registerModule(KotlinModule.Builder().build())          // Kotlin data class 직렬화
            .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
            .activateDefaultTyping(
                LaissezFaireSubTypeValidator.instance,
                ObjectMapper.DefaultTyping.NON_FINAL,
            )
}
```

> **Kotlin idiom 포인트**:
> - `mapOf("a" to x, "b" to y)` — `HashMap.put()` 4 줄이 한 expression.
> - `KotlinModule.Builder().build()` 등록 — data class 의 default 값 / null-safety 가 Jackson 에 인식.

---

## 4. `@Cacheable` 사용

```kotlin
@Service
class ProductQueryService(
    private val products: ProductRepository,
) {

    @Cacheable(cacheNames = ["product:detail"], key = "#productId.value")
    fun getDetail(productId: ProductId): ProductDetailDto =
        products.findById(productId)
            ?.let(::toDto)
            ?: throw BusinessException(ResponseCode.NOT_FOUND, "상품")

    @CacheEvict(cacheNames = ["product:detail"], key = "#productId.value")
    fun onProductChanged(productId: ProductId) {
        // 호출되면 그 키 캐시 삭제
    }

    @CacheEvict(cacheNames = ["product:detail"], allEntries = true)
    fun evictAll() { /* 비상용 */ }

    private fun toDto(product: Product): ProductDetailDto = TODO()
}
```

> **Kotlin idiom 포인트**:
> - `@Cacheable(cacheNames = ["product:detail"])` — Java 의 `{"..."}` array literal 이 Kotlin `[...]`.
> - SpEL key 의 `#productId.value()` → property syntax 면 `#productId.value`.
> - elvis `?:` 로 not-found → throw.

### 4.1 변경 시 자동 invalidate

```kotlin
@Service
class UpdateProductUseCase(
    private val cache: ProductQueryService,
) {
    @Transactional
    fun handle(/* ... */) {
        // ... 변경
        cache.onProductChanged(productId)          // 호출 시점에 evict
        // 또는 도메인 이벤트 listener 가 evict
    }
}
```

> **함정**: 트랜잭션 안에서 evict — 트랜잭션 rollback 시 캐시는 이미 사라짐. 다음 요청이 DB hit. **`@TransactionalEventListener(AFTER_COMMIT)`** 에서 evict 권장.

---

## 5. 직접 RedisTemplate — 더 세밀

```kotlin
@Component
class ProductCache(
    private val redis: RedisTemplate<String, ProductDetailDto>,
) {
    companion object {
        private val TTL: Duration = Duration.ofMinutes(10)
    }

    fun get(id: ProductId): ProductDetailDto? =
        redis.opsForValue().get(key(id))

    fun set(id: ProductId, value: ProductDetailDto) {
        redis.opsForValue().set(key(id), value, TTL)
    }

    fun evict(id: ProductId) {
        redis.delete(key(id))
    }

    private fun key(id: ProductId): String = "shop:cache:product:detail:${id.value}"
}
```

> **Kotlin idiom 포인트**:
> - `Optional<X>` 없이 nullable `X?` 직접 반환.
> - `"shop:cache:product:detail:${id.value}"` string template.

→ `@Cacheable` 보다 명시적 제어. 복잡한 캐시 로직에 적합.

---

## 6. Cache Stampede 방지

같은 키가 만료된 순간 100 요청이 동시에 DB hit → DB 폭주. 3 가지 방어:

### 방어 1 — 분산 락 (singleflight)

```kotlin
@Service
class ProductDetailService(
    private val cache: ProductCache,
    private val repo: ProductRepository,
    private val redisson: RedissonClient,
) {

    fun get(id: ProductId): ProductDetailDto {
        cache.get(id)?.let { return it }

        // miss — 락 잡고 한 번만 DB 조회
        val lock = redisson.getLock("lock:cache-refresh:product:${id.value}")
        return try {
            if (!lock.tryLock(1, 3, TimeUnit.SECONDS)) {
                // 락 못 잡음 — 다른 노드가 채워줄 것. 짧게 대기 후 캐시 재조회.
                Thread.sleep(50)
                return cache.get(id) ?: loadFromDb(id)
            }
            // 락 잡았으면 다시 한 번 캐시 확인 (다른 노드가 채웠을 수도)
            cache.get(id)?.let { return it }

            val fresh = loadFromDb(id)
            cache.set(id, fresh)
            fresh
        } catch (e: InterruptedException) {
            Thread.currentThread().interrupt()
            loadFromDb(id)
        } finally {
            if (lock.isHeldByCurrentThread) lock.unlock()
        }
    }

    private fun loadFromDb(id: ProductId): ProductDetailDto =
        repo.findById(id)?.let(::toDto)
            ?: throw BusinessException(ResponseCode.NOT_FOUND)

    private fun toDto(product: Product): ProductDetailDto = TODO()
}
```

> **Kotlin idiom 포인트**:
> - `cache.get(id)?.let { return it }` — null 체크 + 조기 반환 한 줄.
> - `lock.isHeldByCurrentThread` — Java getter 가 property.

→ N 요청 중 1 만 DB. 나머지는 캐시 또는 대기.

### 방어 2 — Probabilistic Early Expiration (XFetch)

만료 전부터 일부 요청을 무작위로 refresh 하도록 분산.

```kotlin
fun getWithEarlyRefresh(id: ProductId): ProductDetailDto {
    val entry = cache.getWithTtl(id) ?: return loadAndCache(id)

    val remaining = entry.remainingTtl
    val beta = 1.0                                          // tuning param
    val earlyRefreshThreshold = beta * ln(Math.random()) * -1 * 5000   // 평균 5초

    if (remaining.toMillis() < earlyRefreshThreshold) {
        // background refresh (async)
        CompletableFuture.runAsync { loadAndCache(id) }
    }
    return entry.value
}
```

→ 만료 직전 일부 요청만 새로 DB 가져옴. stampede 자연 분산.

> **선택지 — coroutines**: `Dispatchers.IO` 기반 `launch { loadAndCache(id) }` 로 더 가볍게. `kotlinx-coroutines-core` 의존성 필요.

### 방어 3 — Stale-While-Revalidate

만료된 데이터도 잠깐 응답 + 백그라운드 refresh:

```kotlin
fun getStaleWhileRevalidate(id: ProductId): ProductDetailDto {
    val entry = cache.getWithTtl(id) ?: return loadAndCache(id)

    return if (entry.remainingTtl.isPositive) {
        entry.value        // 정상
    } else {
        // 만료됐지만 stale 데이터 응답 + 비동기 refresh
        CompletableFuture.runAsync { loadAndCache(id) }
        entry.value
    }
}
```

→ 사용자 경험 우선. 강한 정합성 X 도메인 (상품 / 게시글) 에 적합.

---

## 7. 무효화 (invalidation) 전략

### 7.1 명시적 evict (위 §4.1)

가장 단순. 변경마다 evict.

### 7.2 TTL 만 (lazy invalidation)

evict 안 함, TTL 만료 기다림. 데이터 일관성 약함 — 단순 read 화면에 OK.

### 7.3 Pub/Sub — 분산 인스턴스 동기화

```
Instance A: UPDATE product → evict 로컬 + Redis PUBLISH "product-evict" id
Instance B/C: SUBSCRIBE "product-evict" → 자기 로컬 캐시도 evict
```

→ **multi-level cache** (local + Redis) 일 때.

### 7.4 캐시 키 versioning

`product:detail:v2:{id}` — 키 자체 prefix 를 V2 로 바꾸면 V1 자동 폐기. 큰 schema 변경 시.

---

## 8. Multi-level Cache (Caffeine + Redis)

```kotlin
implementation("com.github.ben-manes.caffeine:caffeine:3.1.8")
```

```kotlin
@Configuration
class TwoLevelCacheConfig {

    @Bean
    fun cacheManager(connectionFactory: RedisConnectionFactory): CacheManager {
        val caffeine: Caffeine<Any, Any> = Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(1))
        val localManager = CaffeineCacheManager().apply {
            setCaffeine(caffeine)
        }

        val redisManager = redisCacheManager(connectionFactory)

        return CompositeCacheManager(localManager, redisManager)
    }
}
```

→ **L1 = JVM (μs)** + **L2 = Redis (ms)** + **L3 = DB**. 인스턴스 마다 다른 L1 = Pub/Sub invalidation 필요.

---

## 9. Cache key 설계 규칙

```
shop:cache:{domain}:{operation}:{paramHash}

예:
  shop:cache:product:detail:01HZ-PRODUCT-ID
  shop:cache:product:search:hash-of-{brandId,categoryId,priceMin,priceMax,sort,cursor}
  shop:cache:user:profile:01HZ-USER-ID
```

- prefix 통일 (`shop:cache:`) — Redis SCAN / 전체 evict
- domain 분리
- params hash — 검색 캐시는 SHA-256 of (sorted params)
- TTL 도메인 별 (product 10m, user 5m, category 1h)

---

## 10. 측정 / 모니터링

```kotlin
// MetricsCacheConfig.kt
@Bean
fun cacheMetrics(registry: MeterRegistry): CacheManagerCustomizer<RedisCacheManager> =
    CacheManagerCustomizer { manager ->
        manager.cacheNames.forEach { name ->
            manager.getCache(name)?.let {
                GuavaCacheMetrics(name, it).bindTo(registry)
            }
        }
    }
```

지표:
- hit rate / miss rate per cache
- evict count
- key size distribution
- value size (큰 값 = 메모리)

→ Grafana / Prometheus 에서 hit rate < 50% 이면 캐시 의미 없음 (재검토).

---

## 11. 함정 모음

### 함정 1 — 강한 일관성 데이터 캐시
재고 / 잔액 / 결제 = 캐시 X. 1초의 불일치도 사고.

### 함정 2 — 트랜잭션 안 evict
rollback 시 캐시 이미 사라짐. AFTER_COMMIT.

### 함정 3 — `@Cacheable` value `null`
default = null 캐시함 → 다음 요청도 null. `disableCachingNullValues()` 또는 명시적 처리.

### 함정 4 — 큰 객체 캐시
1MB+ 객체 = Redis 메모리 부담 + 네트워크. **DTO 만** + 큰 필드 제외.

### 함정 5 — stampede 미방어
인기 상품 만료 = 1000 요청 동시 DB. **분산 락 또는 stale-while-revalidate**.

### 함정 6 — local cache invalidation 미동기화
인스턴스 A 가 evict 했는데 B 는 아직. **Pub/Sub** 또는 TTL 짧게.

### 함정 7 — 직렬화 형식 변경
v1 의 Java serialization → v2 의 JSON 으로 바꿈 = 옛 캐시 deserialize 실패. **key versioning + flush** 또는 둘 다 지원.

### 함정 8 — Redis 메모리 만수
LRU / LFU eviction policy 명시. `maxmemory-policy allkeys-lru` 권장.

### 함정 9 — 캐시 키에 user-controllable 값
사용자가 무한 다른 키 생성 = 메모리 폭발. 키는 우리가 생성하는 ID 만.

### 함정 10 — `@Cacheable` 의 self-invocation
같은 클래스 메서드 호출 = AOP 우회. 별도 빈 / `@CacheableMethod` 분리.

### 함정 11 — Kotlin 한정: `@Cacheable` 메서드가 `private` / `final`
default 가 `final` 인 Kotlin 클래스 메서드는 `kotlin("plugin.spring")` 없으면 proxy 가 안 됨 → `@Cacheable` 동작 X. **plugin.spring 적용** 또는 명시적 `open fun`.

### 함정 12 — Kotlin 한정: `KotlinModule` 미등록
ObjectMapper 가 Kotlin data class 의 default 값 / nullable 필드를 모름 → null 역직렬화 실패. **`KotlinModule.Builder().build()` 등록 필수**.

---

## 12. 운영 체크리스트

- [ ] hit rate / miss rate 모니터링
- [ ] Redis `maxmemory-policy` 설정
- [ ] 핵심 캐시는 stampede 방어 (락 / SWR / probabilistic)
- [ ] TTL 도메인 별 차등
- [ ] evict 는 AFTER_COMMIT
- [ ] 캐시 schema 변경 시 key versioning
- [ ] 큰 객체 캐시 자제 (DTO 만)
- [ ] Redis 다운 시 fail-over (DB 직접 — 부하 ↑ 수용 또는 회로차단)
- [ ] `KotlinModule` 등록 + `kotlin("plugin.spring")` 적용

---

## 13. 관련

- [[distributed-lock]] — stampede 방지에 사용
- [[../../java-spring/api-design/cache-redis|↗ Java 원본]]
- [[api-design|↑ api-design hub]]
