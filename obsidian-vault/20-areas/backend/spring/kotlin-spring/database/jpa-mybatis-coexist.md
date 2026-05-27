---
title: "JPA + MyBatis 공존 (Kotlin) — 같이 쓰는 표준 패턴"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T15:05:00+09:00
tags:
  - backend
  - kotlin-spring
  - database
  - jpa
  - mybatis
  - coexistence
---

# JPA + MyBatis 공존 (Kotlin) — 같이 쓰는 표준 패턴

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + Kotlin 한정 함정 (plugin / dirty flush / DTO 차이) |

**[[database|↑ database hub]]** · **[[../../java-spring/database/jpa-mybatis-coexist|↗ Java 원본]]**

> 전제 — [[jpa]] / [[mybatis]] 본편 먼저.

---

## 1. 왜 같이 쓰나 / 2. 핵심 원칙 / 5. 같은 테이블 다른 도구 / 7. 트랜잭션 / 8. Recipe 별 분담

(Java 원본 §1, §2, §5, §7, §8 와 동일 — 설계 원칙은 언어 무관)

---

## 3. 설정 — 단일 DataSource + 단일 TransactionManager

```kotlin
// build.gradle.kts
implementation("org.springframework.boot:spring-boot-starter-data-jpa")
implementation("org.mybatis.spring.boot:mybatis-spring-boot-starter:3.0.3")
runtimeOnly("org.postgresql:postgresql")
```

(yaml 설정은 Java 원본 [[../../java-spring/database/jpa-mybatis-coexist#3 설정|§3]] 와 동일)

> **그게 다** — Spring Boot 의 auto-configuration 이 같은 DataSource 를 두 starter 가 공유.

### 3.1 트랜잭션 매니저

```kotlin
@Service
class SignupService(
    private val userJpa: UserJpaRepository,        // JPA
    private val auditMapper: AuditMapper,           // MyBatis
) {

    @Transactional                                  // 둘 다 같은 트랜잭션
    fun signup(user: UserJpaEntity, audit: AuditLogRow) {
        userJpa.save(user)
        auditMapper.insert(audit)
        // 한쪽 실패 시 둘 다 rollback
    }
}
```

> **Kotlin 한정**: `@Transactional` 메서드가 final → proxy X. `kotlin("plugin.spring")` 필수.

---

## 4. 역할 분담 패턴

```kotlin
// 도메인 port
interface UserRepository {                          // CUD + 단순 조회
    fun save(user: User): User
    fun findByEmail(email: Email): User?
}

interface UserQueryRepository {                     // 복잡 검색 / 보고서
    fun search(criteria: UserSearchCriteria, pageable: Pageable): Page<UserSearchResult>
    fun monthlyStats(year: Int): List<UserMonthlyStats>
}

// JPA — 쓰기 + 트랜잭션
@Repository
class JpaUserRepository(
    private val spring: UserJpaRepository,
) : UserRepository {
    // save / findByEmail 구현
    override fun save(user: User): User = TODO()
    override fun findByEmail(email: Email): User? = TODO()
}

// MyBatis — 복잡 read
@Repository
class MyBatisUserQueryRepository(
    private val mapper: UserQueryMapper,
) : UserQueryRepository {

    override fun search(c: UserSearchCriteria, p: Pageable): Page<UserSearchResult> {
        val rows = mapper.search(c, p.offset, p.pageSize)
        val total = mapper.count(c)
        return PageImpl(rows, p, total)
    }

    override fun monthlyStats(year: Int): List<UserMonthlyStats> = mapper.monthlyStats(year)
}
```

→ 도메인은 두 interface 모두 모름. Application service 가 두 repository 모두 주입.

### 4.2 같은 트랜잭션 안의 dirty read 함정

```kotlin
@Transactional
fun doSomething(id: UserId) {
    val user = userJpaRepo.findById(id.value)
        ?: throw BusinessException(ResponseCode.NOT_FOUND)
    user.name = "changed"                            // 객체만 변경 (UPDATE 아직 아님)

    // 같은 트랜잭션에서 MyBatis 로 조회
    val row = userMapper.findById(id.value)          // ⚠️ DB 의 옛 name 이 보임
}
```

**해결**: `em.flush()` 로 UPDATE 즉시 발사 후 MyBatis read:

```kotlin
@Transactional
fun doSomething(id: UserId) {
    val user = userJpaRepo.findById(id.value)
        ?: throw BusinessException(ResponseCode.NOT_FOUND)
    user.name = "changed"
    em.flush()                                       // UPDATE 즉시 발사
    val row = userMapper.findById(id.value)          // 새 name 보임
}
```

**더 나은 규칙**: 같은 트랜잭션 안에서 **같은 테이블의 JPA 쓰기 + MyBatis 읽기 섞지 말 것**.

---

## 6. 같은 테이블 — 무엇을 JPA, 무엇을 MyBatis

(Java 원본 [[../../java-spring/database/jpa-mybatis-coexist#5 같은 테이블, 다른 도구|§5]] 의 분담 표 동일)

| 작업 | JPA | MyBatis |
| --- | --- | --- |
| 사용자 등록 (`POST /users`) | ✅ Aggregate CUD | ❌ |
| 사용자 조회 by id | ✅ Repository.findById | ❌ |
| 사용자 검색 (이름 / 이메일 / 가입일 + paging) | ❌ | ✅ |
| 월별 가입자 통계 | ❌ | ✅ |
| 사용자 soft delete | ✅ Entity 메서드 | ❌ |
| GDPR export (사용자의 모든 데이터 JOIN) | ❌ | ✅ |

---

## 7. 트랜잭션 — JPA 쓰기 + MyBatis 쓰기 같이

```kotlin
@Service
class OrderPlacementService(
    private val orderJpa: OrderJpaRepository,         // JPA (Aggregate 저장)
    private val auditMapper: AuditMapper,             // MyBatis (audit log)
    private val notificationMapper: NotificationMapper,
) {

    @Transactional
    fun placeOrder(order: Order) {
        // 1. JPA 로 Aggregate 저장
        val saved = orderJpa.save(OrderMapper.toEntity(order))

        // 2. MyBatis 로 audit log INSERT
        auditMapper.insert(AuditLogRow(/* ... */))

        // 3. MyBatis 로 알림 INSERT
        notificationMapper.insert(NotificationRow(/* ... */))

        // 하나라도 실패하면 셋 다 rollback
    }
}
```

→ 단일 TransactionManager 가 두 도구 모두 처리. 별도 설정 X.

---

## 9. 함정 모음

(원본 [[../../java-spring/database/jpa-mybatis-coexist#9 함정 모음|§9]] 의 10가지 함정 그대로)

### Kotlin 한정 추가 함정

### 함정 11 — `kotlin("plugin.spring")` 누락 → JPA / MyBatis 둘 다 깨짐
plugin 없으면 final method 라 proxy 못 만들어 `@Transactional` 무효. 두 도구가 같이 깨짐.

### 함정 12 — JPA Entity 는 `class`, MyBatis Row 는 `data class`
JPA `UserJpaEntity` = 일반 class (data class 금지, [[jpa#함정 11]])
MyBatis `UserRow` = data class (reflection 매핑 OK)
같은 패키지에 두면 헷갈리니 **`infrastructure/persistence/jpa/`** vs **`infrastructure/persistence/mybatis/`** 분리.

### 함정 13 — `em.flush()` 의 시점
Kotlin 의 lambda 안에서 `em.flush()` 호출이 transaction proxy 와 동기화 안 될 수 있음. **service method 직접 호출**.

### 함정 14 — Adapter 끼리 같은 도메인 객체를 두 번 mapping
`JpaUserRepository.save(user)` + `MyBatisUserQueryRepository.search(...).map { toDomain(it) }` — 도메인 객체 생성이 두 곳에서. 일관성을 위해 도메인 reconstruct factory (`User.reconstitute(...)`) 한 곳에 모음.

### 함정 15 — `mybatis-spring-boot-starter` 의 Kotlin reflection 의존성
parameter binding 이 `kotlin-reflect` 의존. `runtimeOnly("org.jetbrains.kotlin:kotlin-reflect")` 또는 `implementation`.

---

## 10. 운영 / 디버깅 체크리스트

- [ ] 단일 DataSource (별도 설정 X)
- [ ] 단일 TransactionManager — `JpaTransactionManager`
- [ ] 같은 트랜잭션 안의 JPA 쓰기 + MyBatis 읽기 — `em.flush()`
- [ ] 도메인 reconstruct factory 한 곳
- [ ] `kotlin("plugin.spring")` + `kotlin("plugin.jpa")` 모두
- [ ] `kotlin-reflect` 의존성 (MyBatis parameter binding)
- [ ] JPA / MyBatis 코드를 패키지 분리 (`infrastructure/persistence/jpa` vs `mybatis`)
- [ ] 같은 테이블의 CUD 를 두 도구로 분산 금지

---

## 11. 관련

- [[../../java-spring/database/jpa-mybatis-coexist|↗ Java 원본]]
- [[jpa]] / [[mybatis]]
- [[database|↑ database hub]]
- [[../api-design/api-design#0.5 ORM 정책]]
