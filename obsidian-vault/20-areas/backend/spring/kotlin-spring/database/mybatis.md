---
title: "MyBatis (Kotlin) — 본편 (Spring Boot 3)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T14:55:00+09:00
tags:
  - backend
  - kotlin-spring
  - database
  - mybatis
---

# MyBatis (Kotlin) — 본편 (Spring Boot 3)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + Kotlin data class DTO + null-safety + 함정 |

**[[database|↑ database hub]]** · **[[../../java-spring/database/mybatis|↗ Java 원본]]**

> JPA 와 비교: [[jpa]] / 둘 다: [[jpa-mybatis-coexist]].

---

## 1. 한 줄 정의 / 4. Dynamic SQL / 5. resultMap / 7. cursor pagination / 10. Adapter

(Java 원본 §1, §4, §5, §7, §10 와 동일 — SQL/XML 은 언어 무관)

---

## 2. 표준 의존성

```kotlin
// build.gradle.kts
implementation("org.mybatis.spring.boot:mybatis-spring-boot-starter:3.0.3")
implementation("com.github.pagehelper:pagehelper-spring-boot-starter:2.1.0")
// Kotlin 친화 (Mapper interface 가 open 으로 컴파일되도록 plugin.spring 필수)
```

```yaml
mybatis:
  mapper-locations: classpath:mapper/**/*.xml
  type-aliases-package: com.example.shop.domain
  configuration:
    map-underscore-to-camel-case: true
    default-statement-timeout: 5
    cache-enabled: false
    jdbc-type-for-null: NULL
  type-handlers-package: com.example.shop.infrastructure.persistence.mybatis.typehandler
```

---

## 3. Mapper — Kotlin interface + XML

```kotlin
// infrastructure/persistence/mybatis/user/UserMapper.kt
@Mapper
interface UserMapper {
    fun findById(@Param("id") id: String): UserRow?
    fun findByEmail(@Param("email") email: String): UserRow?
    fun insert(row: UserRow): Int
    fun updatePassword(@Param("id") id: String, @Param("hash") hash: String): Int
    fun delete(@Param("id") id: String): Int
    fun search(criteria: UserSearchCriteria): List<UserRow>
}

// UserRow.kt — data class (DTO)
data class UserRow(
    val id: String,
    val email: String,
    val name: String,
    val passwordHash: String,
    val status: String,
    val createdAt: Instant,
    val updatedAt: Instant,
)
```

> **Kotlin idiom 포인트**:
> - `UserRow?` nullable return — Java `Optional<UserRow>` 와 같음. mybatis-spring-boot-starter 가 null 인식.
> - **data class** DTO — JPA Entity 와 다르게 data class OK (mapper 가 reflection 으로 생성자 호출).
> - **`@Mapper` interface** 는 final 이어도 OK — interface 라 proxy 가능. 단 구현 class 가 있다면 `kotlin("plugin.spring")` 필요.

```xml
<!-- mapper/user/UserMapper.xml — 본문은 Java 원본과 동일 -->
<mapper namespace="com.example.shop.infrastructure.persistence.mybatis.user.UserMapper">
    <resultMap id="UserRowMap" type="com.example.shop.infrastructure.persistence.mybatis.user.UserRow">
        <id     property="id"           column="id"/>
        <result property="email"        column="email"/>
        <result property="name"         column="name"/>
        <result property="passwordHash" column="password_hash"/>
        <result property="status"       column="status"/>
        <result property="createdAt"    column="created_at"/>
        <result property="updatedAt"    column="updated_at"/>
    </resultMap>
    <!-- select/insert/update/delete 본문은 Java 원본과 동일 -->
</mapper>
```

### 3.1 어노테이션 mapper

```kotlin
@Mapper
interface UserMapper {

    @Select("""
        SELECT id, email, name, password_hash AS passwordHash, status,
               created_at AS createdAt, updated_at AS updatedAt
        FROM users WHERE lower(email) = lower(#{email})
    """)
    fun findByEmail(@Param("email") email: String): UserRow?

    @Insert("""
        INSERT INTO users (id, email, password_hash, name, status, created_at, updated_at)
        VALUES (#{id}, #{email}, #{passwordHash}, #{name}, #{status}, #{createdAt}, #{updatedAt})
    """)
    fun insert(row: UserRow): Int
}
```

> **Kotlin idiom**: `"""..."""` raw multi-line string — Java `+ "..." +` 보다 가독성 좋음.

---

## 6. 페이지네이션

```kotlin
data class UserSearchCriteria(
    val email: String?,
    val status: String?,
    val limit: Int,
    val offset: Int,
)

@Select("SELECT ... FROM users WHERE ... LIMIT #{limit} OFFSET #{offset}")
fun search(criteria: UserSearchCriteria): List<UserRow>

@Select("SELECT count(*) FROM users WHERE ...")
fun count(criteria: UserSearchCriteria): Long
```

(PageHelper 사용은 Java 원본 §6.2 와 동일)

---

## 8. typeHandler — 도메인 타입 ↔ DB

```kotlin
// infrastructure/persistence/mybatis/typehandler/EmailTypeHandler.kt
@MappedTypes(Email::class)
@MappedJdbcTypes(JdbcType.VARCHAR)
class EmailTypeHandler : BaseTypeHandler<Email>() {

    override fun setNonNullParameter(
        ps: PreparedStatement, i: Int, parameter: Email, jdbcType: JdbcType,
    ) {
        ps.setString(i, parameter.value)
    }

    override fun getNullableResult(rs: ResultSet, columnName: String): Email? =
        rs.getString(columnName)?.let(::Email)

    override fun getNullableResult(rs: ResultSet, columnIndex: Int): Email? =
        rs.getString(columnIndex)?.let(::Email)

    override fun getNullableResult(cs: CallableStatement, columnIndex: Int): Email? =
        cs.getString(columnIndex)?.let(::Email)
}
```

> **Kotlin idiom**: Java 의 `if (s == null) null else new Email(s)` 4 줄 → `rs.getString(...)?.let(::Email)` 한 줄.

### 8.2 JSONB

```kotlin
@MappedTypes(Map::class)
@MappedJdbcTypes(JdbcType.OTHER)
class JsonbMapTypeHandler : BaseTypeHandler<Map<String, Any?>>() {
    private val json = ObjectMapper().registerKotlinModule()

    override fun setNonNullParameter(
        ps: PreparedStatement, i: Int, v: Map<String, Any?>, t: JdbcType,
    ) {
        val pgo = PGobject().apply {
            type = "jsonb"
            value = json.writeValueAsString(v)
        }
        ps.setObject(i, pgo)
    }

    override fun getNullableResult(rs: ResultSet, columnName: String): Map<String, Any?>? =
        rs.getString(columnName)?.let {
            @Suppress("UNCHECKED_CAST")
            json.readValue(it, Map::class.java) as Map<String, Any?>
        }

    // 나머지 getNullableResult 도 동일
    override fun getNullableResult(rs: ResultSet, columnIndex: Int): Map<String, Any?>? = TODO()
    override fun getNullableResult(cs: CallableStatement, columnIndex: Int): Map<String, Any?>? = TODO()
}
```

> **Kotlin 한정**: `ObjectMapper().registerKotlinModule()` — data class default 값 / nullable 인식.

---

## 9. 트랜잭션

JPA 와 동일 — `@Transactional` 사용.

```kotlin
@Service
class UserService(
    private val users: UserMapper,
) {
    @Transactional
    fun updateName(id: String, newName: String) {
        users.findById(id) ?: throw BusinessException(ResponseCode.NOT_FOUND)
        users.updateName(id, newName)        // 명시적 UPDATE
    }
}
```

> **JPA 와 다른 점**: dirty checking 없음 — **명시적 UPDATE 호출 필수**.
> **Kotlin 한정**: `@Transactional` 메서드가 `final` 이면 proxy 안 됨 → `kotlin("plugin.spring")` 필수.

---

## 10. Repository Adapter — 도메인 격리

```kotlin
// 도메인 port
interface UserRepository {
    fun findByEmail(email: Email): User?
    fun save(user: User): User
}

// MyBatis adapter
@Repository
class MyBatisUserRepository(
    private val mapper: UserMapper,
    private val clock: Clock,
) : UserRepository {

    override fun findByEmail(email: Email): User? =
        mapper.findByEmail(email.value)?.let(::toDomain)

    override fun save(user: User): User {
        val row = toRow(user, Instant.now(clock))
        val existing = mapper.findById(user.id.value) != null
        runCatching {
            if (existing) mapper.update(row) else mapper.insert(row)
        }.onFailure { ex ->
            if (ex is DuplicateKeyException) {
                throw EmailAlreadyExistsException(user.email)
            }
            throw ex
        }
        return user
    }

    private fun toDomain(r: UserRow): User = User.reconstitute(
        id = UserId(r.id),
        email = Email(r.email),
        passwordHash = PasswordHash(r.passwordHash),
        name = r.name,
        status = UserStatus.valueOf(r.status),
        createdAt = r.createdAt,
    )

    private fun toRow(u: User, now: Instant): UserRow = UserRow(
        id = u.id.value,
        email = u.email.value,
        passwordHash = u.passwordHash.value,
        name = u.name,
        status = u.status.name,
        createdAt = u.createdAt,
        updatedAt = now,
    )
}
```

> **Kotlin idiom**: `?.let(::toDomain)` — null 이면 null, 아니면 변환.

---

## 10.5 Recipe 별 MyBatis 적용 (Kotlin)

(Java 원본 §10.5 의 7가지 recipe 별 패턴 동일 — XML 은 그대로, Kotlin mapper interface + Adapter 만 변환.)

### 10.5.6 비관 락 SQL 직접

```kotlin
@Mapper
interface ProductOptionMapper {
    @Select("SELECT id, name, value, extra_price_krw AS extraPriceKrw, stock " +
            "FROM product_options WHERE id = #{id} FOR UPDATE")
    fun findForUpdate(@Param("id") id: String): ProductOptionRow?

    @Update("UPDATE product_options SET stock = stock - #{qty} " +
            "WHERE id = #{id} AND stock >= #{qty}")
    fun decreaseStock(@Param("id") id: String, @Param("qty") qty: Int): Int
}

@Service
class StockService(
    private val options: ProductOptionMapper,
) {
    @Transactional
    fun decreaseStock(id: String, qty: Int) {
        val affected = options.decreaseStock(id, qty)
        if (affected == 0) {
            throw BusinessException(ResponseCode.CONFLICT, "재고 부족")
        }
    }
}
```

→ **rowCount = 0 면 재고 부족** 검사. SELECT 없이 한 쿼리로 검증 + 차감.

### 10.5.8 공통 — Kotlin Adapter 5가지 규칙

(Java 원본 §10.5.8 의 5규칙 그대로 + Kotlin)

1. Row DTO 는 mapper 전용 `data class` — 도메인 (`User`) 이 알지 못함.
2. `@Transactional` 안에서만 mapper 호출.
3. `#{}` 만 사용 — `${}` 는 메타에만.
4. 자식 cascade 는 명시적.
5. `map-underscore-to-camel-case = true` + AS 별칭.
6. **(Kotlin 추가)** `kotlin("plugin.spring")` 적용 — Service / Adapter / Repository class 가 자동 open.

---

## 11. 함정 모음

(원본 [[../../java-spring/database/mybatis#11 함정 모음|§11]] 의 12 함정 그대로)

### Kotlin 한정 추가 함정

### 함정 13 — `@Mapper` interface 의 default method
Kotlin interface 의 default method 는 JVM 의 `static` invokeInterface — MyBatis proxy 가 못 잡음. **abstract 만** 두기.

### 함정 14 — `Long?` vs `Long` return type
SQL `count(*)` 가 `Long` 일지 `Long?` 일지 — mybatis 가 SQL 결과가 null 일 수 있다 판단 시 `Long?` 반환. `count(*)` 는 항상 0+ 이므로 **`Long` non-null** 명시.

### 함정 15 — `data class` parameter binding
`fun insert(row: UserRow)` — MyBatis 가 `#{id}`, `#{email}` 등 property name 으로 binding. data class component 함수 (`component1()`...) 가 아님. **`@Param` 없으면 reflection 으로 property name 검색** — `kotlin-reflect` 의존성 필요.

### 함정 16 — JSON typeHandler 의 `Map<String, Any?>` 타입 erasure
런타임 generic 정보 손실 → `KotlinModule` 등록 + `TypeReference<Map<String, Any?>>` 사용 권장.

### 함정 17 — Kotlin string template `${}` 과 MyBatis `${}` 충돌
`@Select("WHERE x = ${value}")` — Kotlin 이 컴파일 시 `value` evaluate 시도 → 에러. MyBatis 의 `${}` 는 **무조건 `\${}` 또는 `${'$'}{}`** escape.

---

## 12. JPA vs MyBatis — 같은 기능 (Kotlin)

```kotlin
// JPA
@Service
class JpaSignupService(
    private val repo: UserJpaRepository,
) {
    @Transactional
    fun save(user: UserJpaEntity) { repo.save(user) }   // dirty checking
}

// MyBatis
@Service
class MyBatisSignupService(
    private val mapper: UserMapper,
) {
    @Transactional
    fun save(row: UserRow) { mapper.insert(row) }
}
```

→ Kotlin 에선 둘 다 짧음. MyBatis 의 XML 분리는 그대로 trade-off.

---

## 13. 운영 체크리스트

(Java 원본 §13 그대로 + Kotlin)

- [ ] `kotlin("plugin.spring")` 적용 (Mapper 구현체 / Service / Adapter 가 open)
- [ ] `kotlin-reflect` 의존성 (parameter binding)
- [ ] `KotlinModule` ObjectMapper 등록 (JSON typeHandler)
- [ ] `${'$'}{}` 또는 `\${}` 로 MyBatis `${}` escape

---

## 14. 관련

- [[database|↑ database hub]]
- [[../../java-spring/database/mybatis|↗ Java 원본]]
- [[jpa]] — JPA 본편 / 비교
- [[jpa-mybatis-coexist]] — 같이 쓸 때
