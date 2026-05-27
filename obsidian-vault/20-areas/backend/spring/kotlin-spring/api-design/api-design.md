---
title: "Kotlin Spring — API 구현 레시피 Hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T11:00:00+09:00
tags:
  - backend
  - kotlin-spring
  - api-design
  - recipe
  - hub
---

# Kotlin Spring — API 구현 레시피 Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring/api-design hub 미러링 + Kotlin idiom (DSL / data class / sealed) |

**[[../kotlin-spring|↑ Kotlin Spring]]** · **[[../../spring|↑ Spring hub]]** · **[[../../java-spring/api-design/api-design|↗ Java 원본]]**

> 이 폴더는 **Kotlin + Spring Boot 3** 기준으로 실전 프로젝트에 반복 등장하는 API 기능들의 구현 레시피.
> 각 레시피는 **OOP 설계 → DB 선택 → 보안/암호화 → 코드 → 테스트 → 운영 체크리스트** 구조.
> "이 문서만 보고 현장에서 바로 구현" 을 목표.
>
> Java 원본은 [[../../java-spring/api-design/api-design|↗ java-spring]] 에 정리되어 있고, 본 hub 는 동일 구조를 Kotlin 으로 재작성.

---

## 0. 표준 스택

| 영역 | 버전 / 라이브러리 |
| --- | --- |
| Kotlin | **1.9.x** (또는 2.0.x) |
| JVM | **21 LTS** |
| Spring Boot | **3.3.x** |
| Build | Gradle (Kotlin DSL) |
| DB | PostgreSQL 16 + Spring Data JPA / Hibernate 6 |
| 마이그레이션 | Flyway |
| Security | Spring Security 6 + jjwt 0.12 |
| Cache / Redis | Spring Data Redis + Redisson |
| 테스트 | JUnit 5 + MockK + Kotest + Testcontainers |

```kotlin
// build.gradle.kts (공통 베이스)
plugins {
    kotlin("jvm") version "1.9.24"
    kotlin("plugin.spring") version "1.9.24"           // @Component / @Configuration 자동 open
    kotlin("plugin.jpa") version "1.9.24"              // @Entity no-arg 생성자
    id("org.springframework.boot") version "3.3.0"
    id("io.spring.dependency-management") version "1.1.5"
}

java {
    toolchain.languageVersion = JavaLanguageVersion.of(21)
}

kotlin {
    compilerOptions {
        jvmTarget.set(org.jetbrains.kotlin.gradle.dsl.JvmTarget.JVM_21)
        freeCompilerArgs.addAll("-Xjsr305=strict")     // platform type strict
    }
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")

    // Kotlin 한정
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")     // data class 직렬화
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core")          // suspend (선택)
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-reactor")        // reactive 와 bridge (선택)

    runtimeOnly("org.postgresql:postgresql")

    // 보안 / 암호화
    implementation("com.password4j:password4j:1.8.2")
    implementation("io.jsonwebtoken:jjwt-api:0.12.6")
    runtimeOnly("io.jsonwebtoken:jjwt-impl:0.12.6")
    runtimeOnly("io.jsonwebtoken:jjwt-jackson:0.12.6")

    // 유틸 / 외부
    implementation("io.github.resilience4j:resilience4j-spring-boot3:2.2.0")
    implementation("software.amazon.awssdk:s3:2.25.50")
    implementation("org.redisson:redisson-spring-boot-starter:3.27.2")
    implementation("io.azam.ulidj:ulidj:5.2.3")

    testImplementation("org.springframework.boot:spring-boot-starter-test") {
        exclude(module = "mockito-core")                  // MockK 사용
    }
    testImplementation("org.springframework.security:spring-security-test")
    testImplementation("org.testcontainers:junit-jupiter:1.19.7")
    testImplementation("org.testcontainers:postgresql:1.19.7")
    testImplementation("io.mockk:mockk:1.13.10")
    testImplementation("com.ninja-squad:springmockk:4.0.2")
    testImplementation("io.kotest:kotest-runner-junit5:5.8.0")
    testImplementation("io.kotest:kotest-assertions-core:5.8.0")
    testImplementation("io.kotest.extensions:kotest-extensions-spring:1.1.3")
}
```

> **Kotlin idiom 포인트**:
> - `kotlin("plugin.spring")` — Spring 클래스 (final default 인 Kotlin) 를 자동 `open` 으로 컴파일 → CGLIB proxy 호환.
> - `kotlin("plugin.jpa")` — `@Entity` 가 no-arg 생성자 필요한 JPA 와 호환.
> - `jackson-module-kotlin` — data class JSON 직렬화 / null-safe 처리.
> - `-Xjsr305=strict` — Java 의 `@Nullable` / `@NonNull` 을 Kotlin null-safety 로 strict 적용.
> - 테스트는 MockK + Kotest 가 Kotlin DSL 친화 (Mockito 도 가능하지만 final 클래스 문제).

---

## 0.5 ORM 정책 — 3 모드 (JPA only / MyBatis only / 공존)

> **이 vault 의 레시피는 3 ORM 모드 모두 first-class 로 다룬다.**

### 0.5.1 도메인은 ORM 을 모른다 (Port-Adapter)

모든 레시피는 다음 구조:

```
domain/           ← Aggregate / Value Object / Repository interface (port)
   ↑ (의존)
application/      ← UseCase. @Transactional. Repository port 만 사용.
   ↑ (의존)
infrastructure/   ← Repository 구현 = JPA Adapter / MyBatis Adapter / 둘 다
```

→ Application / Domain 코드는 **ORM 바뀌어도 안 변함**. JPA → MyBatis 전환 = infrastructure adapter 만 교체.

### 0.5.2 Recipe 의 §6 "구현" 을 어떻게 읽나

각 recipe 의 §6 는 **JPA Adapter sketch 가 기본** (Kotlin/Spring 진영에서도 dirty checking / 트랜잭션 안전성 덕에 도메인 CUD 의 사실상 표준).

| 내 프로젝트 모드 | 어떻게 읽나 |
| --- | --- |
| **JPA only** | recipe §6 그대로 사용. 추가로 [[../database/jpa]] 의 영속성·N+1·OSIV. |
| **MyBatis only** | recipe §6 의 도메인 / Application / Controller 까지는 그대로. Repository adapter 는 [[../database/mybatis#7. Recipe 별 적용]] 참고. |
| **JPA + MyBatis 공존** | recipe §6 의 JPA adapter 는 CUD. 검색/보고서/통계는 [[../database/jpa-mybatis-coexist#8. Recipe 별 분담]] 의 MyBatis Query Repository 추가. |

### 0.5.3 어떤 모드를 언제

| 시나리오 | 권장 모드 |
| --- | --- |
| 신규 SaaS / 도메인 풍부 / 작은 팀 | **JPA only** |
| 거대 레거시 SI / SQL 자산 / OLAP 비중 큼 | **MyBatis only** |
| **이커머스 / 한국 SaaS 의 흔한 production** | **공존** — JPA (CUD) + MyBatis (검색·통계) |
| 신규 SaaS 인데 일부 보고서가 너무 복잡 | **공존** (시작은 JPA, 보고서만 MyBatis 추가) |

자세한 결정 트리: [[../database/database#2. 의사결정 트리]].

### 0.5.4 Repository Port 의 분리 원칙 (공존 대비)

레시피의 도메인 layer 가 **2개의 port** 를 갖도록 작성:

```kotlin
// domain/{domain}/
interface XRepository {                              // CUD + 단순 조회
    fun findById(id: XId): X?
    fun existsBy...(...): Boolean
    fun save(x: X): X
}
interface XQueryRepository {                         // 복잡 검색 / 보고서 (옵션)
    fun search(c: XSearchCriteria, p: Pageable): Page<XSummary>
    fun stats(...): List<XStats>
}
```

- **JPA only** 모드면 `XQueryRepository` 도 JPA Specification 으로 구현 (또는 안 만듦)
- **공존** 모드면 `XRepository` = JPA / `XQueryRepository` = MyBatis
- **MyBatis only** 면 둘 다 MyBatis 로 구현

→ 처음부터 port 분리해두면 추후 모드 변경 / ORM 추가가 쉬움.

> **Kotlin 한정**: `Optional<X>` 대신 nullable `X?` — Kotlin null-safety 가 컴파일러 단에서 막아주므로 Optional 의 필요성이 줄어듦.

### 0.5.5 코드 표기 규칙

레시피 §6 안에서:
- JPA 코드는 `// JPA Adapter` 헤더로 표시
- 같은 자리에 MyBatis 변형이 필요할 땐 짧게 sketch + "전체는 [[../database/mybatis#X]] 참고"
- 둘 다 완전한 동작 예제는 **레시피에 안 넣음** — `database/` 노트가 본거지

---

## 1. 각 레시피의 표준 목차 (v2)

> **목표**: "이 문서만 보고도 개발자가 기능 하나를 처음부터 끝까지 구현할 수 있다."
> v1 의 "설계 → 구현 → 테스트 → 운영" 흐름에 **전제 / 완료 조건 / 도메인 규칙 / 구현 순서 / 흔한 함정** 을 추가.

| 섹션                  | 내용                                                                              |
| ------------------- | ------------------------------------------------------------------------------- |
| 0. 전제 / 범위          | 적용 상황 / 사용 기술 / 제외 범위 / 과한 적용 기준                                                |
| 1. 무엇을 만드는가         | API 1~2 개 요구사항 / 요청·응답 예시 / **완료 조건 (Acceptance Criteria)**                     |
| 2. 도메인 모델 (OOP)     | Aggregate / Entity / Value Object / Domain Event / **도메인 규칙 / 상태 전이**           |
| 3. 아키텍처 / 의존성 흐름    | Controller → UseCase → Domain → **Port → Adapter** (Persistence / External)    |
| 4. DB 선택 / 스키마      | RDB vs NoSQL 근거 / **조회 패턴** / 인덱스 / 제약 / **정합성 기준**                             |
| 5. 보안 / 인증·인가 / 암호화 | 인증 주체 / 권한 체크 / 민감정보 / 알고리즘 선정 / OWASP 매핑                                       |
| 6. 구현 — Kotlin      | **패키지 구조** / DTO / Application / Controller / JPA Adapter sketch                |
| 7. 트랜잭션 / 예외 / 검증   | `@Transactional` 경계 / 예외 매핑 / Bean Validation / **동시성 / 멱등성**                   |
| 8. 테스트              | **테스트 시나리오 표** / 단위 / 통합 / 실패 케이스                                               |
| 9. 운영 체크리스트         | 배포 / 설정값 / 로그 / 메트릭 / 알림 / 롤백 / 장애 포인트                                         |
| **10. 구현 순서**       | 어떤 파일과 계층부터 구현할지 단계별 순서 (요구사항 → DB → port → adapter → useCase → controller → 테스트) |
| **11. 흔한 함정**       | 자주 하는 실수와 피해야 할 구현 방식 (+ Kotlin 한정 함정)                                          |
| 12. 관련              | 다른 레시피 cross-link                                                               |

### 1.1 각 섹션의 의도

(Java 원본 [[../../java-spring/api-design/api-design#1.1 각 섹션의 의도|§1.1]] 와 동일 — 내용 중복 회피)

### 1.2 분량이 큰 레시피는 폴더로 split

레시피가 한 파일 (~1000 줄) 을 넘으면 폴더로 분할:

```
{recipe}/
├── {recipe}.md              ← 폴더 hub (overview + 흐름 + TOC + 링크)
├── prerequisites.md         ← §0
├── requirements.md          ← §1
├── domain-model.md          ← §2
├── architecture.md          ← §3
├── database.md              ← §4
├── security.md              ← §5
├── implementation.md        ← §6
├── transactions.md          ← §7
├── testing.md               ← §8
├── operations.md            ← §9
├── implementation-order.md  ← §10
└── pitfalls.md              ← §11
```

### 1.3 코드 컨벤션

- 코드 블록 시작 시 **파일 경로 주석** — `// src/main/kotlin/.../User.kt`
- `import` 는 생략 (가독성)
- 패키지는 `com.example.shop` 가정
- DTO 는 **data class** (Kotlin 표준 immutable + copy + equals)
- Domain entity 는 일반 `class` (JPA 와 호환 — data class 회피, [[../common/security-config#함정 14|함정 14]])
- Value Object 는 **data class** 또는 **value class** (`@JvmInline`)
- Repository port = `domain/` interface / 구현체 = `infrastructure/persistence/{jpa,mybatis}/`
- `Optional<X>` 대신 nullable `X?`
- 상태 다이어그램은 텍스트 (`A → B`) 또는 `mermaid` 코드블록

---

## 2. Tier 1 — 모든 프로젝트 거의 항상 필요

### 2.1 인증 / 인가

| 레시피 | 노트 | 핵심 |
| --- | --- | --- |
| **회원가입** | [[signup/signup\|↗ signup]] (예정) | argon2id / email unique / outbox 이메일 인증 |
| **로그인 (JWT)** | [[login-jwt\|↗ login-jwt]] (예정) | access + rotating refresh / 블랙리스트 / Spring Security 필터 |
| **패스워드 리셋** | [[password-reset\|↗ password-reset]] (예정) | 단일 사용 토큰 / 만료 / enumeration 방지 |

---

## 3. Tier 2 — 다음 사이클

### 3.1 인증 / 인가
- **이메일 인증** — verification token / outbox / re-send rate limit
- **소셜 로그인 (OAuth2)** — `spring-boot-starter-oauth2-client`
- **[[two-factor-auth]]** — 2FA / TOTP — Google Authenticator 호환
- **[[rbac-permissions]]** — Role + Permission + `@PreAuthorize` 권한 모델

### 3.2 횡단 관심사
- **[[common-response-envelope|↗ 공통 Response Envelope]]** ✅ — `ApiEnvelope<T>` (success/data/error/meta) + `ApiError` + GlobalExceptionHandler + RequestIdFilter. **모든 신규 프로젝트 표준.**
- **[[rate-limiting]]** — Bucket4j + Redis — IP / user / endpoint 별
- **Audit log** — Hibernate Envers / `@CreatedBy`

### 3.3 캐시 / 성능
- **[[cache-redis]]** — cache-aside / `@Cacheable` / stampede 방지
- **[[distributed-lock]]** — Redisson 기반 분산 락

### 3.4 파일 / 미디어
- **[[file-upload-s3]]** — S3 presigned URL + MIME / 크기 검증

### 3.5 외부 통합 / 실시간
- **[[websocket-stomp]]** — WebSocket + STOMP — 채팅/알림 push
- **[[webhook-send]]** — 서명 (HMAC) + 재시도 + 멱등 보장

### 3.6 검색 / 위치
- **[[elasticsearch-integration]]** — 색인 / 자동완성 / 한글 형태소
- **[[geo-search]]** — 좌표 검색 / 반경 / Redis GEO vs PostGIS

### 3.7 도메인 기능 / UX
- **[[feed-timeline]]** — fan-out 전략 (push / pull / hybrid)
- **[[recommendation]]** — 협업 / 인기도 / 최근 본 상품
- **[[review]]** — 평점 + 베스트 + 신고 + 익명

### 3.8 데이터 가져오기 · 내보내기
- **[[excel-csv-import-export]]** — POI / OpenCSV — 대용량 streaming

---

## 4. 모든 레시피 공통 원칙

### 4.1 OOP / 도메인 모델링

- **풍부한 도메인 모델 (Rich Domain Model)** — Entity 가 동사. `user.changePassword(...)` 가 `userService.changePassword(user, ...)` 보다 우선.
- **불변 Value Object** — `Email`, `Money`, `PasswordHash`. `@JvmInline value class` 또는 `data class`. 생성 시 검증 (init block 또는 secondary constructor).
- **Aggregate 경계** — 트랜잭션 = Aggregate. 간 의존은 ID 참조 + 도메인 이벤트.
- **Domain Event** — 외부 부수효과 (이메일/Kafka) 는 이벤트로 알리고 Application Layer 가 처리.
- **헥사고날 (Port & Adapter)** — domain 은 framework 모름. JPA / Spring 의존은 infrastructure.

> **Kotlin idiom 한 마디**: `sealed class` / `sealed interface` 로 도메인 이벤트 / state machine 을 type-safe 하게 표현. `when` 이 exhaustive 검사를 강제.

### 4.2 DB 선택 기본 의사결정

(Java 원본 [[../../java-spring/api-design/api-design#4.2 DB 선택 기본 의사결정|§4.2]] 와 동일 — 도메인 / 인프라 선택은 언어와 무관)

### 4.3 보안 / 암호화 — 외울 만한 기본

| 용도 | 알고리즘 (2026) | 비고 |
| --- | --- | --- |
| 패스워드 해시 | **Argon2id** (또는 bcrypt cost 12+) | 단방향. salt 내장 |
| 토큰 / 세션 ID | `SecureRandom` 32 bytes → Base64URL | 절대 `Random` X |
| HMAC (webhook 서명) | **HMAC-SHA-256** | 시크릿은 vault |
| 대칭 (DB 컬럼 암호화) | **AES-256-GCM** | nonce 12 bytes 매번 새로 |
| 비대칭 (JWT 서명) | **RS256 / ES256** | HS256 은 단일 서비스용 |
| 데이터 무결성 | **SHA-256** | MD5 / SHA-1 금지 |

> **함정**: SHA-256 / MD5 로 패스워드 해시는 **사고**. 무조건 password-hashing 전용 (Argon2id / bcrypt / scrypt).

상세: [[../../../../security/security|↗ 20-areas/security]]

### 4.4 응답 / 예외 표준

```kotlin
// Envelope — Kotlin sealed result 패턴
data class ApiResponse<T>(val data: T?, val error: ApiError?) {
    companion object {
        fun <T> ok(data: T): ApiResponse<T> = ApiResponse(data, null)
        fun <T> fail(err: ApiError): ApiResponse<T> = ApiResponse(null, err)
    }
}

data class ApiError(
    val code: String,
    val message: String,
    val details: Map<String, Any>? = null,
)
```

→ 본 hub 표준은 [[../common/response-envelope|↗ common/response-envelope]] 의 `CommonResponse<T>` 가 SSoT. 위는 간략 예시.

예외는 `@RestControllerAdvice` 한 곳에서 매핑. 상세는 각 레시피 §7.

### 4.5 트랜잭션 경계

- `@Transactional` 은 **Application Service 메서드** 에 (Controller / Repository X)
- 한 트랜잭션 = **한 Aggregate** 변경 (분산 트랜잭션 회피)
- 외부 호출 (이메일 / PG / Kafka) 은 **트랜잭션 밖** — outbox 또는 `@TransactionalEventListener(AFTER_COMMIT)`

> **Kotlin 한정 함정**: `@Transactional` 메서드가 final (Kotlin default) 이면 Spring proxy 가 실패. `kotlin("plugin.spring")` 적용 시 자동 open — 별도 `open` 키워드 불필요.

### 4.6 입력 검증

- DTO data class 에 `jakarta.validation` 어노테이션 (`@field:Email`, `@field:Size`, custom)
- Controller 에 `@Valid` → `MethodArgumentNotValidException` → `@RestControllerAdvice` 422 매핑
- **도메인 검증** 은 `init { require(...) }` block 또는 secondary constructor 에 (이중 방어)

> **Kotlin 한정**: validation 어노테이션은 `@field:Email` 처럼 target 명시. data class 생성자 파라미터는 default 가 property → field/method/param 중 어디에 붙는지 모호. **`@field:` 명시 필수**.

```kotlin
data class SignupRequest(
    @field:Email val email: String,
    @field:Size(min = 8) val password: String,
)
```

---

## 5. 추천 패키지 구조

```
src/main/kotlin/com/example/shop
├── domain/                     # framework 모름. POKO + 도메인 로직
│   ├── user/                   # User, Email, PasswordHash, UserRepository (interface)
│   ├── order/
│   └── payment/
├── application/                # UseCase. @Transactional 여기서
│   ├── user/
│   └── order/
├── infrastructure/             # JPA / Redis / S3 / 외부 API. Spring 의존
│   ├── persistence/jpa/        # *JpaEntity, *JpaRepository
│   ├── persistence/redis/
│   ├── external/pg/
│   └── external/email/
├── presentation/               # REST controller, DTO (data class)
│   └── api/v1/
└── config/                     # SecurityConfig, JpaConfig
```

도메인 → infrastructure 임포트 금지 (의존성 역전). Repository 는 domain interface, JPA 구현은 infrastructure.

---

## 6. 진행 상태

> 본 hub 는 java-spring 미러링이 점진적으로 진행 중. 우선 단일 파일 13건 (`6.2`) 부터 시작.

### 6.1 폴더 분할 레시피 (대형 도메인 — 예정)

| 레시피 | 노트 | 비고 |
| --- | --- | --- |
| **회원가입** | `signup/signup` (예정) | argon2id + email-verification + password-reset 흡수 |
| **게시판** | `board/board` (예정) | CRUD + 권한 + 첨부 + Soft Delete |
| **상품 / 결제** | `product/product` (예정) | 디지털 + PG + Webhook + Kafka |
| **알림** | `notification/notification` (예정) | FCM / APNs / SES / WebPush 멀티 채널 |
| **채팅** | `chat/chat` (예정) | 카톡 1:1 / 그룹 / 오픈채팅 / 멀티 디바이스 |

### 6.2 단일 파일 레시피 (카테고리별 — 본 PR 범위)

#### 6.2.1 횡단 관심사
| 노트 | 한 줄 |
| --- | --- |
| [[common-response-envelope]] | api-design 안에서 envelope 보조 노트 (본거지는 [[../common/response-envelope]]) |

#### 6.2.2 인증 · 인가
| 노트 | 한 줄 |
| --- | --- |
| [[two-factor-auth]] | 2FA / TOTP — Google Authenticator 호환 |
| [[rbac-permissions]] | Role + Permission + `@PreAuthorize` 권한 모델 |

#### 6.2.3 횡단 관심사 (추가)
| 노트 | 한 줄 |
| --- | --- |
| [[rate-limiting]] | Bucket4j + Redis — IP / user / endpoint 별 |

#### 6.2.4 캐시 · 동시성
| 노트 | 한 줄 |
| --- | --- |
| [[cache-redis]] | cache-aside / `@Cacheable` / stampede 방지 |
| [[distributed-lock]] | Redisson 기반 분산 락 (재고 / 중복 처리) |

#### 6.2.5 파일 · 미디어
| 노트 | 한 줄 |
| --- | --- |
| [[file-upload-s3]] | S3 presigned URL + MIME / 크기 검증 |

#### 6.2.6 실시간 · 외부 통합
| 노트 | 한 줄 |
| --- | --- |
| [[websocket-stomp]] | WebSocket + STOMP — 채팅/알림 push |
| [[webhook-send]] | 서명 (HMAC) + 재시도 + 멱등 보장 |

#### 6.2.7 검색 · 위치
| 노트 | 한 줄 |
| --- | --- |
| [[elasticsearch-integration]] | 색인 / 자동완성 / 한글 형태소 |
| [[geo-search]] | 좌표 검색 / 반경 / Redis GEO vs PostGIS |

#### 6.2.8 도메인 기능 / UX
| 노트 | 한 줄 |
| --- | --- |
| [[feed-timeline]] | fan-out 전략 (push / pull / hybrid) |
| [[recommendation]] | 협업 / 인기도 / 최근 본 상품 |
| [[review]] | 평점 + 베스트 + 신고 + 익명 |

#### 6.2.9 데이터 가져오기 · 내보내기
| 노트 | 한 줄 |
| --- | --- |
| [[excel-csv-import-export]] | POI / OpenCSV — 대용량 streaming |

---

## 7. 관련

- [[../kotlin-spring|↑ Kotlin Spring]]
- [[../../java-spring/api-design/api-design|↗ Java 원본]]
- [[../../../../database/database|↗ database hub]]
- [[../../../../security/security|↗ security hub]]
- [[../common/response-envelope|↗ kotlin-spring common/response-envelope]]
- [[../common/security-config|↗ kotlin-spring common/security-config]]
