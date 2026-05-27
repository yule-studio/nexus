---
title: "03. Domain Entity + JPA Repository + Flyway"
kind: guide
status: current
created_at: 2026-05-27T10:30:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, kotlin-spring, guidelines, jpa, flyway, entity]
---

# 03. Domain Entity + JPA Repository + Flyway

**[[guidelines|↑ Hub]]** ← [[02-project-structure]] · → 다음 [[04-controller-and-validation]]

> User / Memo entity 를 `open class` + `var` 로 정의 + Spring Data JPA Repository + Flyway 마이그레이션.

---

## 1. JPA 가 무엇인가 (요약)

- **JPA** = Java Persistence API. Java 진영의 ORM 표준 인터페이스.
- **Hibernate** = JPA 의 구현체 (Spring Boot 기본).
- **Spring Data JPA** = Hibernate 위의 Spring 추상화. Repository interface 만 만들면 SQL 자동.

```
[코드 — Kotlin]                         [JPA 표준]                  [DB]
class User { ... }       ←→    Hibernate (EntityManager)    ←→    PostgreSQL
↓
fun save(user: User)    →  INSERT INTO users (...) VALUES (...)
fun findById(1L)         →  SELECT ... FROM users WHERE id = 1
```

→ **Spring Data JPA** = Repository interface + 메서드 이름 / `@Query` → SQL 자동 생성.

---

## 2. Kotlin + JPA 의 4가지 함정 (먼저 알자)

### 함정 1 — JPA Entity 를 `data class` 로
**❌ 절대 X**. `equals` / `hashCode` 가 모든 필드 → `@OneToMany` lazy collection 이 즉시 로딩됨. **일반 `class`** 로.

### 함정 2 — `class User` 만 쓰면 `final` (Kotlin default)
Hibernate 의 lazy proxy 가 subclass 못 만들어 → `MethodNotFoundException`. **`kotlin("plugin.jpa")` plugin** 이 자동 `open` 처리. (이미 [[01-installation-and-hello-world#4.5|01 §4.5]] 에서 적용.)

### 함정 3 — `@Entity` no-arg 생성자 없음
Hibernate 가 reflection 으로 인스턴스 생성 시 `InstantiationException`. **`kotlin("plugin.jpa")` plugin** 이 자동 합성.

### 함정 4 — `val` 만 쓰면 dirty checking 동작 X
`var` (변경 가능) 필드여야 setter 동작. immutable 강제하면 `update_entity` 부분만 `var` 권장.

→ **결론**: build.gradle.kts 의 plugin 3개 + Entity 는 일반 class + `var` 필드.

---

## 3. User Entity

```kotlin
// src/main/kotlin/com/example/memo/user/User.kt
package com.example.memo.user

import jakarta.persistence.*
import org.hibernate.annotations.CreationTimestamp
import org.hibernate.annotations.UpdateTimestamp
import java.time.Instant

enum class UserStatus { ACTIVE, INACTIVE, DELETED }

@Entity
@Table(name = "users")
class User(

    @Column(nullable = false, unique = true, length = 254)
    var email: String,

    @Column(name = "password_hash", nullable = false, length = 255)
    var passwordHash: String,

    @Column(nullable = false, length = 100)
    var name: String,
) {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null
        protected set                                  // JPA 만 set 가능

    @Enumerated(EnumType.STRING)                        // ⚠️ ORDINAL X — enum 순서 바뀌면 사고
    @Column(nullable = false, length = 30)
    var status: UserStatus = UserStatus.ACTIVE

    @CreationTimestamp
    @Column(name = "created_at", nullable = false, updatable = false)
    lateinit var createdAt: Instant
        protected set

    @UpdateTimestamp
    @Column(name = "updated_at", nullable = false)
    lateinit var updatedAt: Instant
        protected set

    fun changePassword(newHash: String) {
        passwordHash = newHash
    }

    fun deactivate() {
        status = UserStatus.INACTIVE
    }

    override fun toString(): String = "User(id=$id, email=$email)"
}
```

> **핵심 패턴**:
> - **primary constructor** = 생성 시 필수 필드 (email / passwordHash / name).
> - **body 안 var** = 자동 생성 / 변경 가능 필드 (id / status / timestamps).
> - **`protected set`** = JPA / Hibernate 만 변경. 외부 코드는 read-only.
> - **`@Enumerated(EnumType.STRING)`** = enum 을 VARCHAR 로 저장 (안전).
> - **`@CreationTimestamp` / `@UpdateTimestamp`** = Hibernate 가 자동 채움.
> - **method (`changePassword`, `deactivate`)** = 도메인 행동을 entity 안에 (rich domain).

### 3.1 왜 `Long?` (nullable id)?

- 새 entity (DB 저장 전) 면 id = `null`.
- `save()` 후 DB 가 IDENTITY 로 생성한 id 가 들어감.
- 따라서 entity 생애 동안 id 가 nullable.

---

## 4. Memo Entity (1:N — User has many Memos)

```kotlin
// src/main/kotlin/com/example/memo/memo/Memo.kt
package com.example.memo.memo

import com.example.memo.user.User
import jakarta.persistence.*
import org.hibernate.annotations.CreationTimestamp
import org.hibernate.annotations.UpdateTimestamp
import java.time.Instant

@Entity
@Table(name = "memos", indexes = [
    Index(name = "ix_memos_owner_created", columnList = "owner_id, created_at DESC"),
])
class Memo(

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "owner_id", nullable = false)
    var owner: User,

    @Column(nullable = false, length = 200)
    var title: String,

    @Column(nullable = false, columnDefinition = "TEXT")
    var content: String = "",
) {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    var id: Long? = null
        protected set

    @CreationTimestamp
    @Column(name = "created_at", nullable = false, updatable = false)
    lateinit var createdAt: Instant
        protected set

    @UpdateTimestamp
    @Column(name = "updated_at", nullable = false)
    lateinit var updatedAt: Instant
        protected set

    fun isOwnedBy(userId: Long): Boolean = owner.id == userId

    fun update(newTitle: String?, newContent: String?) {
        newTitle?.let { title = it }
        newContent?.let { content = it }
    }
}
```

> **`@ManyToOne(fetch = FetchType.LAZY)`** = Memo 조회 시 User 자동 fetch 안 함 (필요 시 명시). N+1 회피.
> **`@JoinColumn(name = "owner_id")`** = FK 컬럼명.

### 4.1 1:N 의 반대편 — `@OneToMany` 안 함?

User 에 `@OneToMany(mappedBy = "owner") val memos: List<Memo>` 도 가능하지만 **권장하지 않음**:
- 사용자 객체 로딩 시 모든 memo 가 fetch 위험 (lazy 라도 attribute 접근 시 N+1).
- 보통 Memo 쪽에서 `findByOwnerId(...)` 로 조회.

**규칙**: 자주 조회하는 방향 (Memo → User) 만 매핑. 반대편 컬렉션은 Repository 로.

---

## 5. Spring Data JPA Repository

```kotlin
// src/main/kotlin/com/example/memo/user/UserRepository.kt
package com.example.memo.user

import org.springframework.data.jpa.repository.JpaRepository

interface UserRepository : JpaRepository<User, Long> {

    fun findByEmail(email: String): User?

    fun existsByEmail(email: String): Boolean
}
```

```kotlin
// src/main/kotlin/com/example/memo/memo/MemoRepository.kt
package com.example.memo.memo

import org.springframework.data.domain.Page
import org.springframework.data.domain.Pageable
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Query
import org.springframework.data.repository.query.Param

interface MemoRepository : JpaRepository<Memo, Long> {

    fun findByOwnerIdOrderByCreatedAtDesc(ownerId: Long, pageable: Pageable): Page<Memo>

    @Query("select m from Memo m where m.owner.id = :ownerId and m.id = :id")
    fun findByIdAndOwnerId(@Param("id") id: Long, @Param("ownerId") ownerId: Long): Memo?
}
```

> **메서드 이름 magic** — Spring Data JPA 가 `findByXxxAndYyy`, `existsByXxx`, `countByXxx` 등을 자동 query 로 변환.

### 5.1 자주 쓰는 메서드 명명

| 메서드 이름 | 자동 생성 SQL |
| --- | --- |
| `findById(id)` | `SELECT ... WHERE id = ?` (기본 제공) |
| `findByEmail(email)` | `SELECT ... WHERE email = ?` |
| `findByEmailAndStatus(email, status)` | `WHERE email = ? AND status = ?` |
| `findByEmailIgnoreCase(email)` | `WHERE lower(email) = lower(?)` |
| `findByNameContaining(name)` | `WHERE name LIKE '%?%'` |
| `findByCreatedAtGreaterThan(date)` | `WHERE created_at > ?` |
| `findByStatusIn(statuses)` | `WHERE status IN (?, ?, ?)` |
| `existsByEmail(email)` | `SELECT 1 ... WHERE email = ?` (boolean) |
| `countByStatus(status)` | `SELECT count(*) ... WHERE status = ?` |
| `deleteByEmail(email)` | `DELETE ... WHERE email = ?` |
| `findFirst10ByOrderByCreatedAtDesc()` | `... ORDER BY created_at DESC LIMIT 10` |

### 5.2 명시 JPQL — `@Query`

복잡한 query 는 `@Query` (JPQL):

```kotlin
@Query("""
    select m from Memo m
    where m.owner.id = :ownerId
      and (:keyword is null or m.title like %:keyword% or m.content like %:keyword%)
    order by m.createdAt desc
""")
fun searchByOwner(
    @Param("ownerId") ownerId: Long,
    @Param("keyword") keyword: String?,
    pageable: Pageable,
): Page<Memo>
```

`@Query(value = "SELECT ...", nativeQuery = true)` = 진짜 SQL (DB 종속).

### 5.3 EntityGraph — N+1 회피

```kotlin
@EntityGraph(attributePaths = ["owner"])              // owner 같이 fetch (JOIN)
@Query("select m from Memo m where m.id = :id")
fun findWithOwner(@Param("id") id: Long): Memo?
```

→ Memo 조회 시 owner 도 한 query 로. 그렇지 않으면 owner 접근 시점에 추가 SELECT (N+1).

### 5.4 락 — 비관 락

```kotlin
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select m from Memo m where m.id = :id")
fun findByIdForUpdate(@Param("id") id: Long): Memo?
```

`SELECT ... FOR UPDATE` — row lock. 재고 차감 같은 critical section.

---

## 6. Flyway — DB 마이그레이션

**Flyway** = SQL 파일을 버전 순서로 DB 에 적용 + `flyway_schema_history` 테이블로 추적.

### 6.1 파일 위치

```
src/main/resources/db/migration/
├── V1__init.sql                            # 첫 placeholder
├── V2__create_users.sql
└── V3__create_memos.sql
```

> **파일명 규칙**: `V<버전>__<설명>.sql`. underscore 2개.

### 6.2 V2__create_users.sql

```sql
-- src/main/resources/db/migration/V2__create_users.sql
CREATE TABLE users (
    id              BIGSERIAL PRIMARY KEY,
    email           VARCHAR(254) NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    name            VARCHAR(100) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'ACTIVE',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_users_email ON users (lower(email));
```

### 6.3 V3__create_memos.sql

```sql
-- src/main/resources/db/migration/V3__create_memos.sql
CREATE TABLE memos (
    id              BIGSERIAL PRIMARY KEY,
    owner_id        BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title           VARCHAR(200) NOT NULL,
    content         TEXT NOT NULL DEFAULT '',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_memos_owner_created ON memos (owner_id, created_at DESC);
```

### 6.4 실행

`./gradlew bootRun` 시 Spring Boot 가 자동으로 Flyway 실행 (V1 → V2 → V3 순서). 첫 실행 후 `flyway_schema_history` 테이블 확인:

```bash
psql postgresql://memo:memo@localhost:5432/memo -c "SELECT * FROM flyway_schema_history"
# installed_rank | version | description | type | script | checksum | installed_by | installed_on | execution_time | success
# ---------------+---------+-------------+------+-------- +----------+--------------+--------------+----------------+---------
#       1       |    1    |   init      | SQL  | V1__... |   xxx    |     memo     |    ...       |       10       |   t
#       2       |    2    | create users|...   |...      |          |              |              |                |
```

### 6.5 운영 규칙

- ✅ V<N> 파일은 한 번 commit 후 **절대 수정 X**. 다음 V<N+1> 로 추가만.
- ✅ rollback 필요 시 V<N+1>__revert.sql 로 down migration.
- ❌ `spring.jpa.hibernate.ddl-auto = update` 운영에 X. 항상 `validate`.
- ❌ `ddl-auto = create-drop` = 모든 데이터 날아감. dev only.

---

## 7. Service — JPA Repository 사용

```kotlin
// src/main/kotlin/com/example/memo/memo/MemoService.kt
package com.example.memo.memo

import com.example.memo.common.ForbiddenException
import com.example.memo.common.NotFoundException
import com.example.memo.user.User
import org.springframework.data.domain.PageRequest
import org.springframework.data.domain.Sort
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
@Transactional(readOnly = true)                        // class 단 default — read only
class MemoService(
    private val memoRepository: MemoRepository,
) {
    fun listByOwner(ownerId: Long, page: Int, size: Int): List<Memo> {
        val pageable = PageRequest.of(page, size, Sort.by(Sort.Direction.DESC, "createdAt"))
        return memoRepository.findByOwnerIdOrderByCreatedAtDesc(ownerId, pageable).content
    }

    fun getOwned(memoId: Long, ownerId: Long): Memo {
        val memo = memoRepository.findById(memoId).orElse(null)
            ?: throw NotFoundException("메모")
        if (!memo.isOwnedBy(ownerId)) {
            throw ForbiddenException("본인의 메모만 조회 가능합니다.")
        }
        return memo
    }

    @Transactional                                      // 쓰기는 명시
    fun create(owner: User, title: String, content: String): Memo {
        val memo = Memo(owner = owner, title = title, content = content)
        return memoRepository.save(memo)
    }

    @Transactional
    fun update(memoId: Long, ownerId: Long, newTitle: String?, newContent: String?): Memo {
        val memo = getOwned(memoId, ownerId)
        memo.update(newTitle, newContent)
        return memo                                     // dirty checking — save() 불필요
    }

    @Transactional
    fun delete(memoId: Long, ownerId: Long) {
        val memo = getOwned(memoId, ownerId)
        memoRepository.delete(memo)
    }
}
```

> **`@Transactional`** 핵심:
> - class 단 default `readOnly = true` → 쓰기 메서드만 `@Transactional` 명시.
> - dirty checking — `memo.update(...)` 호출만으로 UPDATE SQL 자동 발사 (commit 시점).
> - private function 에 붙이면 동작 X (Spring proxy 가 public 만 처리).

---

## 8. JPA 의 영속성 컨텍스트 (1차 캐시)

```kotlin
@Transactional
fun example() {
    val memo1 = memoRepository.findById(1L).get()       // SQL: SELECT ...
    val memo2 = memoRepository.findById(1L).get()       // SQL: 없음! (1차 캐시 hit)
    println(memo1 === memo2)                            // true — 같은 instance
}
```

→ 같은 트랜잭션 안에서 같은 ID 조회 = SQL 한 번. **동일 instance 보장** (`===`).

---

## 9. 자주 막히는 함정 (JPA / Flyway)

### 함정 1 — `data class User : ...` JPA Entity
🔥 **절대 X**. 일반 `class`.

### 함정 2 — `lateinit` 사용 시 `Long?` PK 와 충돌
`@Id lateinit var id: Long` = 컴파일 에러 (primitive 는 lateinit X). **`var id: Long? = null`**.

### 함정 3 — `@OneToMany` 의 cascade / orphanRemoval
잘못 쓰면 DELETE / UPDATE 대량 발사. 작은 collection 만 cascade. **본 가이드는 cascade 안 씀** (Memo 가 직접 owner FK).

### 함정 4 — JPQL 의 entity 명 vs table 명
JPQL = entity (`select m from Memo m`). nativeQuery = table (`SELECT * FROM memos m`). 헷갈리지 말 것.

### 함정 5 — `findById(id).get()` vs `findById(id).orElse(null)`
`get()` 은 `Optional` 이 비어있으면 `NoSuchElementException`. `orElseThrow { NotFoundException() }` 권장.

### 함정 6 — Flyway V<N>.sql checksum 변경
한 번 적용한 V<N> 수정 시 startup 실패 (`Migration checksum mismatch`). **새 V<N+1>** 만 추가.

### 함정 7 — Flyway 가 dev 환경에서 V1 부터 다시 실행
없는 history 면 baseline 잡음. 옛 DB 에 적용 시 `baseline-on-migrate: true` 임시 사용. 운영은 항상 `false`.

### 함정 8 — `@Transactional` 메서드를 같은 클래스 안 다른 메서드에서 호출
Spring proxy 우회 → 트랜잭션 동작 X. **다른 클래스의 메서드 또는 `self.method(...)` 패턴**.

### 함정 9 — `ddl-auto: validate` 가 새 컬럼 못 잡음
Entity 에 새 컬럼 추가 → DB 에 없음 → startup fail (`Schema-validation: missing column`). **Flyway V<N+1> 작성 + migrate**.

### 함정 10 — `open class` 누락
plugin.jpa 가 자동 처리하지만 plugin 없으면 직접 `open class User { ... }`.

---

## 10. 다음

✅ Entity + Repository + Flyway 끝.

→ [[04-controller-and-validation]] — `@RestController` + DTO + Bean Validation + 글로벌 예외 처리
