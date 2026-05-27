---
title: "Spring Data JPA / Hibernate (Kotlin) — 본편"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T14:40:00+09:00
tags:
  - backend
  - kotlin-spring
  - database
  - jpa
  - hibernate
---

# Spring Data JPA / Hibernate (Kotlin) — 본편

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + plugin.jpa + open class entity + Kotlin null-safety 함정 |

**[[database|↑ database hub]]** · **[[../../java-spring/database/jpa|↗ Java 원본]]**

> 가장 흔한 사고는 **N+1**, **OSIV**, **Kotlin entity 의 final/no-arg 함정**.

---

## 1. JPA 와 Hibernate / Spring Data JPA / 2. 의존성

(Java 원본 §1, §2 와 동일 — yaml 설정 + 표준 의존성 동일)

### 2.1 Kotlin 한정 plugin 적용

```kotlin
// build.gradle.kts
plugins {
    kotlin("jvm") version "1.9.24"
    kotlin("plugin.spring") version "1.9.24"            // @Component / @Service / @Transactional → open
    kotlin("plugin.jpa") version "1.9.24"               // @Entity → no-arg 생성자 + open
    kotlin("kapt") version "1.9.24"                     // QueryDSL 사용 시
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    runtimeOnly("org.postgresql:postgresql")
    // ...
}

// allopen 추가 — JPA 클래스가 final 이면 lazy proxy 가 안 됨
allOpen {
    annotation("jakarta.persistence.Entity")
    annotation("jakarta.persistence.MappedSuperclass")
    annotation("jakarta.persistence.Embeddable")
}
```

> **핵심**: `kotlin("plugin.jpa")` 가 자동으로 `@Entity` 클래스에 no-arg 생성자 (JPA reflection 용) + open 처리.

---

## 3. Entity — 매핑 기본

```kotlin
// infrastructure/persistence/jpa/user/UserJpaEntity.kt
@Entity
@Table(name = "users")
class UserJpaEntity(
    @Id
    @Column(length = 26)
    val id: String,                                  // ULID — 직접 발급 (auto X)

    @Column(nullable = false, length = 254, unique = true)
    var email: String,

    @Column(name = "password_hash", nullable = false, length = 255)
    var passwordHash: String,

    @Column(nullable = false, length = 100)
    var name: String,

    @Enumerated(EnumType.STRING)                     // ⚠️ ORDINAL X
    @Column(nullable = false, length = 30)
    var status: UserStatus,

    @Version
    @Column(nullable = false)
    var version: Long = 0L,                          // 낙관 락

    @Column(name = "created_at", nullable = false, updatable = false)
    val createdAt: Instant,

    @Column(name = "updated_at", nullable = false)
    var updatedAt: Instant,
)
```

> **Kotlin idiom + JPA 함정 포인트**:
> - **`data class` 절대 금지** — `equals`/`hashCode` 가 모든 필드 → lazy `@OneToMany` collection 이 즉시 로딩됨. 일반 `class`.
> - **`val` vs `var`** — JPA 가 변경하는 필드는 `var`. `id`, `createdAt` 같은 immutable 은 `val`.
> - **no-arg 생성자** — `kotlin("plugin.jpa")` 가 합성. 직접 `protected constructor()` 안 써도 됨.
> - **final class 함정** — JPA lazy proxy 가 subclass 만들기 위해 entity 가 `open` 필요. plugin 이 자동 처리.

### 3.1 ID 전략

(Java 원본 §3.1 와 동일 — ULID 직접 발급 권장)

### 3.2 enum / JSON / Embedded

```kotlin
@Enumerated(EnumType.STRING)
var status: OrderStatus = OrderStatus.PENDING

@JdbcTypeCode(SqlTypes.JSON)                         // Hibernate 6.2+
@Column(columnDefinition = "jsonb")
var metadata: Map<String, Any?> = emptyMap()

@Embedded
var shipping: ShippingAddress? = null
```

> **Kotlin 한정 함정**: `Map<String, Object>` 가 Kotlin 에선 `Map<String, Any?>` — Jackson + Hibernate JSON type 이 nullable value 인식. `KotlinModule` 등록 ([[../api-design/cache-redis#3 설정]] 참고).

### 3.3 컬렉션 / 관계

```kotlin
@OneToMany(
    mappedBy = "product",
    cascade = [CascadeType.ALL],                    // Kotlin array literal
    orphanRemoval = true,
    fetch = FetchType.LAZY,
)
@BatchSize(size = 100)
val options: MutableList<ProductOptionJpaEntity> = mutableListOf()

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "brand_id", nullable = false)
var brand: BrandJpaEntity? = null                    // nullable — JPA proxy 가 null 가능성
```

> **Kotlin 한정 함정**:
> - `cascade = [CascadeType.ALL]` — Java `{CascadeType.ALL}` 가 Kotlin 의 `[...]`.
> - `options` 가 `val MutableList` — collection reference 는 final, 안의 원소는 mutable. JPA dirty checking 호환.
> - `@ManyToOne` 의 `nullable = false` 여도 Kotlin 에선 `var brand: BrandJpaEntity?` 로 — proxy 생성 시점에 일시 null. `lateinit` 도 가능하지만 Hibernate 호환성 주의.

---

## 4. Repository — Spring Data JPA

```kotlin
interface UserJpaRepository :
    JpaRepository<UserJpaEntity, String>,
    JpaSpecificationExecutor<UserJpaEntity> {

    fun findByEmail(email: String): UserJpaEntity?
    fun existsByEmail(email: String): Boolean

    @Query("from UserJpaEntity u where lower(u.email) = lower(:email)")
    fun findByEmailIgnoreCase(@Param("email") email: String): UserJpaEntity?

    @Query(value = "SELECT * FROM users WHERE lower(email) = lower(:email)", nativeQuery = true)
    fun findByEmailNative(@Param("email") email: String): UserJpaEntity?

    @EntityGraph(attributePaths = ["roles", "profile"])
    override fun findById(id: String): Optional<UserJpaEntity>

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select u from UserJpaEntity u where u.id = :id")
    fun findByIdForUpdate(@Param("id") id: String): UserJpaEntity?

    @Modifying
    @Query("update UserJpaEntity u set u.status = :status where u.id = :id")
    fun updateStatus(@Param("id") id: String, @Param("status") status: UserStatus): Int
}
```

> **Kotlin idiom 포인트**:
> - `Optional<X>` 대신 nullable `X?` — Spring Data JPA 가 자동 인식 (3.0+).
> - `@EntityGraph(attributePaths = ["roles", "profile"])` — array literal.
> - `findById` override 시에만 `Optional<X>` 명시 필요 (parent interface signature 따라야).

### 4.1 메서드 명명 규칙 / 4.2 Specification / 4.3 Projection

(Java 원본 §4.1, §4.2, §4.3 와 동일 — DTO projection 은 data class 로 자연스러움)

```kotlin
// Data class projection (Hibernate 6.2+ 가 자동 매핑)
data class UserSummaryDto(val id: String, val email: String, val name: String)

@Query("select new com.example.shop.UserSummaryDto(u.id, u.email, u.name) from UserJpaEntity u")
fun findSummariesDto(): List<UserSummaryDto>

// interface projection 도 가능
interface UserSummary {
    val id: String
    val email: String
    val name: String
}

@Query("select u.id as id, u.email as email, u.name as name from UserJpaEntity u")
fun findSummaries(): List<UserSummary>
```

---

## 5. 영속성 컨텍스트 (Persistence Context)

(Java 원본 §5 와 동일 — JPA 의 dirty checking / 1차 캐시 / lazy proxy 는 언어 무관)

### 5.1 dirty checking 함정 (Kotlin 변환)

```kotlin
@Transactional
fun changeName(id: UserId, newName: String) {
    val user = userRepo.findById(id.value) ?: throw BusinessException(ResponseCode.NOT_FOUND)
    user.name = newName                              // OK — UPDATE 발사됨
    // userRepo.save(user) 호출 안 해도 됨
}

fun changeName(id: UserId, newName: String) {       // ⚠️ @Transactional 없음
    val user = userRepo.findById(id.value) ?: return
    user.name = newName                              // detached state — 변경 안 됨
}
```

→ **`@Transactional` 메서드는 `final` 면 안 됨** — `kotlin("plugin.spring")` 자동 처리 (위 §2.1).

### 5.2 `save` / `flush` / `clear`

(Java 원본 §5.2, §5.3 와 동일 — Kotlin 코드도 같은 패턴)

```kotlin
@Transactional
fun importMany(data: List<UserData>) {
    data.forEachIndexed { i, d ->
        em.persist(UserJpaEntity(/* ... */))
        if (i % 50 == 0) {
            em.flush()
            em.clear()
        }
    }
}
```

---

## 6. 트랜잭션

```kotlin
@Transactional                       // 기본 — REQUIRED
fun update(/* ... */) { /* ... */ }

@Transactional(readOnly = true)      // 읽기 — Hibernate dirty checking 끔
fun get(/* ... */): Detail = TODO()

@Transactional(propagation = Propagation.REQUIRES_NEW)
fun logAudit(/* ... */) { /* ... */ }

@Transactional(isolation = Isolation.REPEATABLE_READ)
fun delicate(/* ... */) { /* ... */ }
```

> **Kotlin 한정 함정**: `@Transactional` 메서드가 final default 라서 proxy 안 만들어짐 → 어노테이션 무시. **`kotlin("plugin.spring")` 필수**.

(Java 원본 §6.1, §6.2 와 동일 — readOnly 효과 / propagation 동작은 동일)

---

## 7. fetch 전략 / N+1

(Java 원본 [[../../java-spring/database/jpa#7 fetch 전략 / N+1|§7]] 와 동일 — fetch join / @EntityGraph / @BatchSize 모두 동일)

```kotlin
// JOIN FETCH
@Query("""
    select distinct p from ProductJpaEntity p
      left join fetch p.options
      where p.id = :id
""")
fun findWithOptions(@Param("id") id: String): ProductJpaEntity?

// @EntityGraph
@EntityGraph(attributePaths = ["options"])
fun findWithOptionsGraph(id: String): ProductJpaEntity?
```

---

## 8. 비관 락 / 낙관 락

```kotlin
// 낙관 락 — @Version
@Version var version: Long = 0L

// 비관 락 — @Lock
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select p from ProductOptionJpaEntity p where p.id = :id")
fun findByIdForUpdate(@Param("id") id: String): ProductOptionJpaEntity?
```

(Java 원본 §8 와 동일 — 사용 시점 / OptimisticLockingFailureException 처리는 동일)

---

## 9. 도메인 ↔ JPA 분리 패턴

도메인 entity 와 JPA entity 를 분리해 헥사고날 도메인 보호.

```kotlin
// domain/user/User.kt (POKO — Plain Old Kotlin Object)
class User private constructor(
    val id: UserId,
    var email: Email,
    var passwordHash: PasswordHash,
    /* ... */
) {
    companion object {
        fun register(/* ... */): User = User(/* ... */)
    }

    fun changePassword(newHash: PasswordHash) { passwordHash = newHash }
}

// infrastructure/persistence/jpa/user/UserMapper.kt
object UserMapper {
    fun toEntity(u: User): UserJpaEntity = UserJpaEntity(
        id = u.id.value,
        email = u.email.value,
        passwordHash = u.passwordHash.value,
        /* ... */
    )

    fun toDomain(e: UserJpaEntity): User = User.reconstruct(
        id = UserId(e.id),
        email = Email(e.email),
        /* ... */
    )
}

// infrastructure/persistence/jpa/user/JpaUserRepository.kt
@Repository
class JpaUserRepository(
    private val jpa: UserJpaRepository,
) : UserRepository {

    override fun findById(id: UserId): User? =
        jpa.findById(id.value).map(UserMapper::toDomain).orElse(null)

    override fun save(user: User): User =
        UserMapper.toDomain(jpa.save(UserMapper.toEntity(user)))
}
```

> **Kotlin idiom**: `object UserMapper` (singleton) 으로 stateless mapper — `companion object` 도 OK.

---

## 10. OSIV (Open Session In View)

(Java 원본 [[../../java-spring/database/jpa#10 OSIV|§10]] 의 OSIV 설명 동일 — **`spring.jpa.open-in-view: false`** 강제)

---

## 11. 함정 모음

(Java 원본 §11 의 모든 함정 그대로)

### Kotlin 한정 추가 함정

### 함정 11 — JPA Entity 를 `data class`
`equals`/`hashCode` 가 모든 필드 → `@OneToMany` collection 의 lazy proxy 가 즉시 로딩됨 (equals 평가). **일반 `class` + `open class` (plugin.jpa 가 자동 적용)**.

### 함정 12 — `kotlin("plugin.jpa")` 미적용
no-arg 생성자 없음 → Hibernate `InstantiationException`. **plugin 적용** 또는 `@NoArg` 직접.

### 함정 13 — `kotlin("plugin.spring")` 미적용 → `@Transactional` 동작 X
final method 에 proxy 못 만들어 dirty checking / propagation 모두 무효. **plugin 필수**.

### 함정 14 — `var name: String` 의 `nullable = false` + Kotlin null-safety 불일치
DB 는 NOT NULL, Kotlin entity 는 nullable → load 시 nullable 인식. **`var name: String` (non-null)** 명시.

### 함정 15 — `@ManyToOne(LAZY)` 의 nullable 매핑
`var brand: Brand?` 가 lazy proxy 의 일시 null 과 충돌. `lateinit` 도 entity field 에선 부적합 (val 강제). Kotlin 권장은 **nullable + null-check** 또는 `by lazy` 직접 구현.

### 함정 16 — `Optional<X>` vs `X?` 혼용
Spring Data 3.0+ 는 nullable return type 직접 지원. **`X?` 통일**, `Optional` 은 override 시에만.

### 함정 17 — Hibernate 6 의 `JdbcTypeCode(JSON)` + Kotlin map
`Map<String, Object>` 가 Kotlin 에선 `Map<String, Any?>` — `KotlinModule` 미등록 시 직렬화 깨짐.

### 함정 18 — Companion object 의 `static` 인식
Java 에서 `User.register(...)` 호출 시 `User.Companion.register(...)` — `@JvmStatic` 으로 boxing 회피 권장.

---

## 12. 운영 / 디버깅 체크리스트

- [ ] `open-in-view: false`
- [ ] `kotlin("plugin.spring")` + `kotlin("plugin.jpa")` 둘 다
- [ ] Entity 는 일반 class (data class 아님)
- [ ] `@Transactional` 메서드의 method visibility (private 안 됨)
- [ ] Optional 대신 nullable 통일
- [ ] N+1 — `@EntityGraph` / fetch join / `@BatchSize`
- [ ] `KotlinModule` ObjectMapper 등록 (JSON type 매핑)
- [ ] Hikari pool / max_connections (위 §2)
- [ ] Flyway + ddl-auto: validate ([[flyway]])

---

## 13. 관련

- [[../../java-spring/database/jpa|↗ Java 원본]]
- [[mybatis]] — 보고서 / 복잡 쿼리
- [[jpa-mybatis-coexist]] — 둘 다
- [[flyway]] — 마이그레이션
- [[database|↑ database hub]]
