---
title: "Java Spring — API 구현 레시피 Hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T10:00:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - hub
---

# Java Spring — API 구현 레시피 Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | hub + Tier 1 (auth) 3개 레시피 — Java 21 + Spring Boot 3 |
| v.1.1.0 | 2026-05-20 | engineering-agent/tech-lead | 공통 Response Envelope 표준 추가 (Tier 2 §3.2, source: codwithyc/spring-sandbox) |

**[[../java-spring|↑ Java Spring]]** · **[[../../spring|↑ Spring hub]]**

> 이 폴더는 **Java + Spring Boot 3** 기준으로 실전 프로젝트에 반복 등장하는 API 기능들의 구현 레시피.
> 각 레시피는 **OOP 설계 → DB 선택 → 보안/암호화 → 코드 → 테스트 → 운영 체크리스트** 구조.
> "이 문서만 보고 현장에서 바로 구현" 을 목표.
>
> Kotlin 변형은 [[../../kotlin-spring/kotlin-spring|↗ kotlin-spring]] 에 추후 누적. 본 hub 는 Java 전용.

---

## 0. 표준 스택

| 영역 | 버전 / 라이브러리 |
| --- | --- |
| Java | **21 LTS** |
| Spring Boot | **3.3.x** |
| Build | Gradle (Kotlin DSL) 또는 Maven |
| DB | PostgreSQL 16 + Spring Data JPA / Hibernate 6 |
| 마이그레이션 | Flyway |
| Security | Spring Security 6 + jjwt 0.12 |
| Cache / Redis | Spring Data Redis + Redisson |
| 테스트 | JUnit 5 + Mockito + Testcontainers |

```kotlin
// build.gradle.kts (공통 베이스)
plugins {
    java
    id("org.springframework.boot") version "3.3.0"
    id("io.spring.dependency-management") version "1.1.5"
}

java {
    toolchain.languageVersion = JavaLanguageVersion.of(21)
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

    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
    testImplementation("org.testcontainers:junit-jupiter:1.19.7")
    testImplementation("org.testcontainers:postgresql:1.19.7")
}
```

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

각 recipe 의 §6 는 **JPA Adapter sketch 가 기본** (Java/Spring 진영에서 가장 흔하고, dirty checking / 트랜잭션 안전성 덕에 도메인 CUD 의 사실상 표준).

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

```java
// domain/{domain}/
public interface XRepository {                      // CUD + 단순 조회
    Optional<X> findById(XId id);
    boolean existsBy...(...);
    X save(X x);
}
public interface XQueryRepository {                 // 복잡 검색 / 보고서 (옵션)
    Page<XSummary> search(XSearchCriteria c, Pageable p);
    List<XStats> stats(...);
}
```

- **JPA only** 모드면 `XQueryRepository` 도 JPA Specification 으로 구현 (또는 안 만듦)
- **공존** 모드면 `XRepository` = JPA / `XQueryRepository` = MyBatis
- **MyBatis only** 면 둘 다 MyBatis 로 구현

→ 처음부터 port 분리해두면 추후 모드 변경 / ORM 추가가 쉬움.

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
| 6. 구현 — Java        | **패키지 구조** / DTO / Application / Controller / JPA Adapter sketch                |
| 7. 트랜잭션 / 예외 / 검증   | `@Transactional` 경계 / 예외 매핑 / Bean Validation / **동시성 / 멱등성**                   |
| 8. 테스트              | **테스트 시나리오 표** / 단위 / 통합 / 실패 케이스                                               |
| 9. 운영 체크리스트         | 배포 / 설정값 / 로그 / 메트릭 / 알림 / 롤백 / 장애 포인트                                         |
| **10. 구현 순서**       | 어떤 파일과 계층부터 구현할지 단계별 순서 (요구사항 → DB → port → adapter → useCase → controller → 테스트) |
| **11. 흔한 함정**       | 자주 하는 실수와 피해야 할 구현 방식                                                           |
| 12. 관련              | 다른 레시피 cross-link                                                               |

### 1.1 각 섹션의 의도

- **§0 전제 / 범위** — "이 레시피를 언제 적용하면 좋고, 언제 과하다" 를 명시. 단순 CRUD 에 Aggregate + Domain Event 다 적용하면 과함.
- **§1 완료 조건** — API spec 보다 "성공 시 어떤 상태가 변경되는가 / 실패 시 어떤 예외가 발생하는가 / 동일 요청 반복 시 어떻게 처리되는가" 가 더 중요.
- **§2 도메인 규칙 / 상태 전이** — Entity 이름보다 "**언제 무엇이 가능하고 불가능한가**". `PENDING → APPROVED → COMPLETED`, `COMPLETED → CANCELED 불가` 같은 다이어그램 또는 텍스트.
- **§3 Port / Adapter 명시** — Domain 의 Repository / External 호출이 어떻게 추상화되고 (port), 어떻게 구현되는지 (adapter). 헥사고날 풀어쓰기.
- **§4 조회 패턴 / 정합성 기준** — 인덱스보다 먼저 "단건 / 목록 / 검색 / 정렬 어떤 패턴", "Unique / FK / Soft Delete 무엇을 보장하나".
- **§5 인증·인가 분리** — "암호화" 보다 "이 endpoint 는 누가 호출 가능한가 (anonymous / authenticated / ROLE / 본인 리소스만)" 가 자주 누락. 인가 누락이 가장 흔한 사고.
- **§6 패키지 구조** — `domain/ application/ infrastructure/ presentation/` 의 폴더 트리. 문서만 보고 파일 생성 가능.
- **§7 동시성 / 멱등성** — "같은 요청 두 번 보내면?", "같은 사용자 동시 요청?", "Unique 제약 / Lock / Idempotency-Key" 중 무엇이 필요한가.
- **§8 테스트 시나리오 표** — `| 케이스 | 입력 | 기대 결과 | 테스트 종류 |`. 정상 + 중복 + 권한 없음 + 잘못된 입력 + 동시성 — 빠뜨리기 어렵게.
- **§9 로그·메트릭·롤백** — 단순 "배포 체크" 가 아니라 "어떤 메트릭 / 알림 / 롤백 가능 여부 / 환경변수".
- **§10 구현 순서** — 설계 문서 끝났을 때 "어디부터 만들지" 막히는 게 가장 흔함. 단계별 to-do.
- **§11 흔한 함정** — Controller 에 비즈니스 로직, Entity 를 Response 로 반환, 단순 CRUD 에 Aggregate 과도 적용 등 — 짧은 don'ts.

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

→ 첫 split 예: [[signup/signup|↗ signup]] (folder).

### 1.3 코드 컨벤션

- 코드 블록 시작 시 **파일 경로 주석** — `// src/main/java/.../User.java`
- `import` 는 생략 (가독성)
- 패키지는 `com.example.shop` 가정
- DTO 는 **record** 우선 (Java 14+ 표준 immutable)
- Repository port = `domain/` interface / 구현체 = `infrastructure/persistence/{jpa,mybatis}/`
- 상태 다이어그램은 텍스트 (`A → B`) 또는 `mermaid` 코드블록

---

## 2. Tier 1 — 모든 프로젝트 거의 항상 필요

### 2.1 인증 / 인가

| 레시피 | 노트 | 핵심 |
| --- | --- | --- |
| **회원가입** | [[signup\|↗ signup]] | argon2id / email unique / outbox 이메일 인증 |
| **로그인 (JWT)** | [[login-jwt\|↗ login-jwt]] | access + rotating refresh / 블랙리스트 / Spring Security 필터 |
| **패스워드 리셋** | [[password-reset\|↗ password-reset]] | 단일 사용 토큰 / 만료 / enumeration 방지 |

---

## 3. Tier 2 — 다음 사이클 (작성 예정)

### 3.1 인증 / 인가
- **이메일 인증** — verification token / outbox / re-send rate limit
- **소셜 로그인 (OAuth2)** — `spring-boot-starter-oauth2-client`
- **2FA / TOTP** — Google Authenticator 호환
- **RBAC / 권한 모델** — Role + Permission + `@PreAuthorize`

### 3.2 횡단 관심사
- **[[common-response-envelope|↗ 공통 Response Envelope]]** ✅ — `ApiEnvelope<T>` (success/data/error/meta) + `ApiError` (code/messageKey/fieldErrors) + GlobalExceptionHandler + RequestIdFilter. **모든 신규 프로젝트 표준.**
- **페이지네이션·검색·표준 응답** — cursor vs offset / Specification / `@RestControllerAdvice`
- **Rate limiting** — Bucket4j + Redis
- **Audit log** — Hibernate Envers / `@CreatedBy`

### 3.3 이커머스 / 트랜잭션
- **결제 (PG + 멱등성 + Webhook)** — `Idempotency-Key` / state machine / webhook 서명
- **주문·재고 (동시성)** — 낙관 락 vs 비관 락 / 분산 락
- **쿠폰 / 할인 / 포인트**
- **재고 예약 (장바구니 hold)**

### 3.4 캐시 / 성능
- **Redis 캐시** — cache-aside / `@Cacheable` / stampede 방지
- **분산 락 (Redisson)**
- **N+1 회피** — fetch join / `@EntityGraph`

### 3.5 파일 / 미디어
- **파일 업로드 (S3 presigned)**
- **이미지 리사이즈 / 썸네일**

### 3.6 비동기 / 메시징
- **이메일/SMS 발송 (transactional outbox)**
- **Kafka producer / consumer**
- **스케줄링** — `@Scheduled` + ShedLock

### 3.7 실시간
- **WebSocket / STOMP**
- **SSE**

### 3.8 외부 통합
- **WebClient + 재시도 / 회로차단** — Resilience4j
- **Webhook 송신** — 서명 + 재시도

---

## 4. 모든 레시피 공통 원칙

### 4.1 OOP / 도메인 모델링

- **풍부한 도메인 모델 (Rich Domain Model)** — Entity 가 동사. `user.changePassword(...)` 가 `userService.changePassword(user, ...)` 보다 우선.
- **불변 Value Object (record)** — `Email`, `Money`, `PasswordHash`. 생성 시 검증.
- **Aggregate 경계** — 트랜잭션 = Aggregate. 간 의존은 ID 참조 + 도메인 이벤트.
- **Domain Event** — 외부 부수효과 (이메일/Kafka) 는 이벤트로 알리고 Application Layer 가 처리.
- **헥사고날 (Port & Adapter)** — domain 은 framework 모름. JPA / Spring 의존은 infrastructure.

### 4.2 DB 선택 기본 의사결정

| 시나리오 | 권장 | 이유 |
| --- | --- | --- |
| 사용자 / 주문 / 결제 등 도메인 핵심 | **PostgreSQL** | ACID + unique constraint + JSONB |
| 세션 / 캐시 / 토큰 / 분산 락 | **Redis** | μs / TTL / atomic |
| 검색 / 자동완성 / 로그 분석 | **Elasticsearch** | 역색인 / 분석기 |
| 이벤트 / 비동기 / 트래킹 | **Kafka** | append-only / replay |
| 파일 / 이미지 / 동영상 | **S3** | object store, presigned URL |
| 시계열 / 메트릭 | **Timescale / Prometheus** | 컬럼 / 압축 |

자세한 비교: [[../../../../database/database|↗ database hub]]

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

```java
// Envelope
public record ApiResponse<T>(T data, ApiError error) {
    public static <T> ApiResponse<T> ok(T data) { return new ApiResponse<>(data, null); }
    public static <T> ApiResponse<T> fail(ApiError err) { return new ApiResponse<>(null, err); }
}
public record ApiError(String code, String message, Map<String, Object> details) {
    public ApiError(String code, String message) { this(code, message, null); }
}
```

예외는 `@RestControllerAdvice` 한 곳에서 매핑. 상세는 각 레시피 §7.

### 4.5 트랜잭션 경계

- `@Transactional` 은 **Application Service 메서드** 에 (Controller / Repository X)
- 한 트랜잭션 = **한 Aggregate** 변경 (분산 트랜잭션 회피)
- 외부 호출 (이메일 / PG / Kafka) 은 **트랜잭션 밖** — outbox 또는 `@TransactionalEventListener(AFTER_COMMIT)`

### 4.6 입력 검증

- DTO 에 `jakarta.validation` 어노테이션 (`@Email`, `@Size`, custom)
- Controller 에 `@Valid` → `MethodArgumentNotValidException` → `@RestControllerAdvice` 422 매핑
- **도메인 검증** 은 record compact constructor / Entity 생성 시점에도 (이중 방어)

---

## 5. 추천 패키지 구조

```
com.example.shop
├── domain/                     # framework 모름. POJO + 도메인 로직
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
├── presentation/               # REST controller, DTO (record)
│   └── api/v1/
└── config/                     # SecurityConfig, JpaConfig
```

도메인 → infrastructure 임포트 금지 (의존성 역전). Repository 는 domain interface, JPA 구현은 infrastructure.

---

## 6. 진행 상태

> 레시피는 **분량 / 도메인 복잡도** 에 따라 두 형태로 관리한다.
> - **6.1 폴더 분할** — 거대한 도메인. §0~§11 가 각 파일.
> - **6.2 단일 파일** — 한 파일 (≤1000 줄) 로 충분한 횡단·기능 레시피. **카테고리별** 로 묶어 navigation 한다.

### 6.1 폴더 분할 레시피 (대형 도메인)

| 레시피 | 노트 | 비고 |
| --- | --- | --- |
| **회원가입** | [[signup/signup]] | argon2id + email-verification + password-reset 흡수 |
| **게시판** | [[board/board]] | CRUD + 권한 + 첨부 + Soft Delete |
| **상품 / 결제** | [[product/product]] | 디지털 + PG + Webhook + Kafka (F10+) |
| **알림** | [[notification/notification]] | FCM / APNs / SES / WebPush 멀티 채널 |
| **채팅** | [[chat/chat]] | 카톡 1:1 / 그룹 / 오픈채팅 / 멀티 디바이스 |

### 6.2 단일 파일 레시피 (카테고리별)

#### 6.2.1 인증 · 인가
| 노트 | 한 줄 |
| --- | --- |
| [[two-factor-auth]] | 2FA / TOTP — Google Authenticator 호환 |
| [[rbac-permissions]] | Role + Permission + `@PreAuthorize` 권한 모델 |

#### 6.2.2 횡단 관심사
| 노트 | 한 줄 |
| --- | --- |
| [[rate-limiting]] | Bucket4j + Redis — IP / user / endpoint 별 |

#### 6.2.3 캐시 · 동시성
| 노트 | 한 줄 |
| --- | --- |
| [[cache-redis]] | cache-aside / `@Cacheable` / stampede 방지 |
| [[distributed-lock]] | Redisson 기반 분산 락 (재고 / 중복 처리) |

#### 6.2.4 파일 · 미디어
| 노트 | 한 줄 |
| --- | --- |
| [[file-upload-s3]] | S3 presigned URL + MIME / 크기 검증 |

#### 6.2.5 실시간 · 외부 통합
| 노트 | 한 줄 |
| --- | --- |
| [[websocket-stomp]] | WebSocket + STOMP — 채팅/알림 push |
| [[webhook-send]] | 서명 (HMAC) + 재시도 + 멱등 보장 |

#### 6.2.6 검색 · 위치
| 노트 | 한 줄 |
| --- | --- |
| [[elasticsearch-integration]] | 색인 / 자동완성 / 한글 형태소 |
| [[geo-search]] | 좌표 검색 / 반경 / Redis GEO vs PostGIS |

#### 6.2.7 도메인 기능 / UX
| 노트 | 한 줄 |
| --- | --- |
| [[feed-timeline]] | fan-out 전략 (push / pull / hybrid) |
| [[recommendation]] | 협업 / 인기도 / 최근 본 상품 |
| [[review]] | 평점 + 베스트 + 신고 + 익명 |

#### 6.2.8 데이터 가져오기 · 내보내기
| 노트 | 한 줄 |
| --- | --- |
| [[excel-csv-import-export]] | POI / OpenCSV — 대용량 streaming |

### 6.3 미작성 / 흡수됨

| 항목 | 상태 |
| --- | --- |
| `login-jwt` | 📝 계획 — Tier 1 (§2.1) 작성 예정 |
| `password-reset` | ✅ 흡수 → [[signup/signup]] (signup/implementation/password-reset-impl) |
| `email-verification` | ✅ 흡수 → [[signup/signup]] (signup/implementation/email-verification-*) |
| `oauth2-social-login` | 📝 계획 — Tier 2 (§3.1) |
| ~~`chat-realtime`~~ | ✅ 흡수 → [[chat/chat]] |

---

## 7. 관련

- [[../java-spring|↑ Java Spring]]
- [[../../../../database/database|↗ database hub]]
- [[../../../../security/security|↗ security hub]]
