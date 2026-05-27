---
title: "위치 기반 검색 (당근 스타일) — Kotlin"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T12:50:00+09:00
tags:
  - backend
  - kotlin-spring
  - api-design
  - recipe
  - geo
  - postgis
---

# 위치 기반 검색 (당근 스타일) — Kotlin

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + Kotlin extension distance label |

**[[api-design|↑ api-design hub]]** · **[[../../java-spring/api-design/geo-search|↗ Java 원본]]**

> 📐 공통: [[../common/response-envelope]] · [[../common/security-config]].

---

## 1. 무엇을 만드는가 / 2. 도구 비교 / 3. DB 스키마 / 4. 반경 검색 SQL / 8. privacy / 9. 함정

(Java 원본 §1, §2, §3, §4 SQL, §8 정책, §9 함정 그대로 — DB / SQL / 정책은 언어 무관)

---

## 4. 반경 검색 Repository (Kotlin)

```kotlin
interface ItemNearbyRepository : JpaRepository<ItemJpaEntity, String> {

    @Query(
        value = """
            SELECT i.id AS id, i.title AS title, i.price_krw AS priceKrw,
                   ST_Distance(i.coordinate,
                     ST_GeogFromText('POINT(' || :lng || ' ' || :lat || ')')) AS distanceM
            FROM items i
            WHERE i.status = 'SELLING'
              AND ST_DWithin(i.coordinate,
                    ST_GeogFromText('POINT(' || :lng || ' ' || :lat || ')'), :radiusM)
            ORDER BY i.coordinate <->
                     ST_GeogFromText('POINT(' || :lng || ' ' || :lat || ')')
            LIMIT :limit
        """,
        nativeQuery = true,
    )
    fun findNearby(
        @Param("lat") lat: Double,
        @Param("lng") lng: Double,
        @Param("radiusM") radiusM: Int,
        @Param("limit") limit: Int,
    ): List<ItemNearbyView>
}
```

> **함정**: `ST_DWithin` 이 인덱스 활용 → **반드시 ST_DWithin** 으로 필터링, `ST_Distance` 는 SELECT / ORDER BY 만.

---

## 5. 동네 인증 (Onboarding)

```kotlin
@Service
class VerifyNeighborhoodUseCase(
    private val userLocations: UserLocationRepository,
    private val neighborhoods: NeighborhoodRepository,
    private val clock: Clock,
) {

    @Transactional
    fun handle(userId: UserId, lat: Double, lng: Double): NeighborhoodInfo {
        // 1. 좌표가 어느 동네에 속하는지
        val neighborhood = neighborhoods.findContaining(lng, lat)
            ?: throw BusinessException(ResponseCode.NOT_FOUND, "해당 위치의 동네를 찾을 수 없습니다.")

        // 2. user_locations upsert
        userLocations.upsert(userId, neighborhood.id, lng, lat, Instant.now(clock))

        return NeighborhoodInfo(
            id = neighborhood.id.value,
            name = neighborhood.name,
            sido = neighborhood.sido,
            sigungu = neighborhood.sigungu,
        )
    }
}

interface NeighborhoodRepository : JpaRepository<NeighborhoodEntity, String> {
    @Query(
        value = """
            SELECT * FROM neighborhoods
            WHERE ST_Contains(boundary::geometry,
                    ST_GeomFromText('POINT(' || :lng || ' ' || :lat || ')', 4326))
            LIMIT 1
        """,
        nativeQuery = true,
    )
    fun findContaining(@Param("lng") lng: Double, @Param("lat") lat: Double): NeighborhoodEntity?
}
```

---

## 6. Redis Geo (옵션)

```kotlin
@Service
class RedisGeoIndex(
    private val redis: RedisTemplate<String, String>,
) {

    fun addItem(id: ItemId, lng: Double, lat: Double) {
        redis.opsForGeo().add("items:geo", Point(lng, lat), id.value)
    }

    fun nearby(lng: Double, lat: Double, radiusKm: Double, limit: Int): List<NearbyItem> {
        val circle = Circle(Point(lng, lat), Distance(radiusKm, Metrics.KILOMETERS))
        val args = RedisGeoCommands.GeoRadiusCommandArgs.newGeoRadiusArgs()
            .includeDistance().includeCoordinates().sortAscending().limit(limit.toLong())

        return redis.opsForGeo().radius("items:geo", circle, args)
            ?.content
            ?.map { r ->
                NearbyItem(
                    id = r.content.name,
                    distanceM = r.distance.value,
                    lng = r.content.point.x,
                    lat = r.content.point.y,
                )
            }
            .orEmpty()
    }

    fun remove(id: ItemId) {
        redis.opsForGeo().remove("items:geo", id.value)
    }
}

@Component
class ItemGeoSyncListener(
    private val redisGeo: RedisGeoIndex,
) {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun on(event: ItemCreated) { redisGeo.addItem(event.itemId, event.lng, event.lat) }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun on(event: ItemDeleted) { redisGeo.remove(event.itemId) }
}
```

> **Kotlin idiom**: `?.content?.map { }.orEmpty()` — null → empty list chain.

---

## 7. Controller

```kotlin
@RestController
@RequestMapping("/api/v1")
@PreAuthorize("isAuthenticated()")
class LocationController(
    private val verifyNeighborhood: VerifyNeighborhoodUseCase,
    private val items: ItemQueryService,
) {

    @PostMapping("/me/location")
    fun setLocation(
        @Valid @RequestBody req: SetLocationRequest,
        auth: Authentication,
    ): ResponseEntity<CommonResponse<NeighborhoodInfo>> {
        val info = verifyNeighborhood.handle(UserId(auth.name), req.lat, req.lng)
        return CommonResponse.success(ResponseCode.OK, info, "동네 인증 완료").toResponseEntity()
    }

    @GetMapping("/items/nearby")
    fun nearby(
        @RequestParam @DecimalMin("33") @DecimalMax("43") lat: Double,
        @RequestParam @DecimalMin("124") @DecimalMax("132") lng: Double,
        @RequestParam(defaultValue = "5000") @Min(100) @Max(20000) radiusM: Int,
        @RequestParam(defaultValue = "20") @Min(1) @Max(100) limit: Int,
    ): ResponseEntity<CommonResponse<List<ItemNearbyView>>> =
        CommonResponse.success(
            ResponseCode.OK,
            items.findNearby(lat, lng, radiusM, limit),
            "조회 성공",
        ).toResponseEntity()
}

data class SetLocationRequest(
    @field:DecimalMin("33") @field:DecimalMax("43") val lat: Double,
    @field:DecimalMin("124") @field:DecimalMax("132") val lng: Double,
)
```

> 한국 좌표 범위 (lat 33~43, lng 124~132) 로 입력 검증.

---

## 8. 응답 distance label — extension

```kotlin
data class ItemNearbyView(
    val id: String,
    val title: String,
    val priceKrw: Long,
    val distanceM: Double,
) {
    val distanceLabel: String
        get() = when {
            distanceM < 100  -> "100m 이내"
            distanceM < 1000 -> "${(distanceM / 100).roundToInt() * 100}m"
            else             -> "%.1fkm".format(distanceM / 1000)
        }
}
```

> **Kotlin idiom**: computed property `val distanceLabel: String get() = when { }` — Java 의 instance method 가 property.

---

## 9. 함정 모음

(원본 [[../../java-spring/api-design/geo-search#9 함정 모음|§9]] 의 10가지 함정 그대로)

### Kotlin 한정 추가 함정

### 함정 11 — `Distance` / `Metrics` import 충돌
Spring Data 의 `org.springframework.data.geo.Distance` 와 Kotlin stdlib (없음) / java.time 의 `Duration` 헷갈림. 명시적 import.

### 함정 12 — Geo radius result nullable
`redis.opsForGeo().radius(...)` 는 nullable `GeoResults<...>?`. **`?.content?.map { }.orEmpty()`** 패턴.

### 함정 13 — `@field:DecimalMin` target 누락
data class 파라미터에 `@DecimalMin` 만 쓰면 어느 element 에 붙는지 모호. `@field:` 명시.

---

## 10. 운영 체크리스트

- [ ] PostGIS extension 설치
- [ ] `geography(POINT, 4326)` 사용
- [ ] GIST 인덱스
- [ ] `ST_DWithin` 으로 필터
- [ ] 좌표 범위 입력 검증 (`@field:DecimalMin`)
- [ ] 응답엔 동네 + 거리 라벨 (좌표 X)
- [ ] 동네 인증 정기 재확인 (2개월)
- [ ] Redis Geo (hot) ↔ PostGIS (진실) 동기화 job

---

## 11. 관련

- [[../../java-spring/api-design/geo-search|↗ Java 원본]]
- [[cache-redis]] — Redis Geo hot index
- [[../common/security-config]]
- [[api-design|↑ api-design hub]]
