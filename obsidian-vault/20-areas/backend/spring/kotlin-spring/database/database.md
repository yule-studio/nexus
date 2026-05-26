---
title: "Kotlin Spring — Data Access / ORM Hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T14:25:00+09:00
tags:
  - backend
  - kotlin-spring
  - database
  - orm
  - hub
---

# Kotlin Spring — Data Access / ORM Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + Kotlin idiom (Exposed / JOOQ-Kotlin / Hibernate Reactive) |

**[[../kotlin-spring|↑ Kotlin Spring]]** · **[[../../java-spring/database/database|↗ Java 원본]]**

> **레시피 (`api-design/`) 에는 Repository port (도메인 interface) 까지만**.
> 구체 ORM 구현은 이 폴더의 노트가 본거지. JPA / MyBatis / 둘 다 / Exposed / JOOQ-Kotlin.
> DB 자체 운영은 [[../../../../database/database|↗ 20-areas/database]].

---

## 1. Kotlin Spring 의 Data Access 6가지

| 도구 | 한 줄 | 강점 | 약점 |
| --- | --- | --- | --- |
| **Spring Data JPA / Hibernate** | 객체-관계 매핑 (ORM) | 도메인 모델링 / 영속성 컨텍스트 | N+1 / OSIV / Kotlin entity 의 final/no-arg 함정 |
| **MyBatis** | SQL 직접 + 결과 매핑 | SQL 그대로 / 복잡 쿼리 | DTO 양산 / Kotlin null-safety 와의 conflict |
| **JOOQ (Kotlin DSL)** | 타입 안전 SQL DSL | 컴파일 검증 / data class 직접 매핑 | 라이선스 / 학습 |
| **JdbcTemplate / NamedParameterJdbcTemplate** | 가장 얇은 wrapper | 단순 / 명확 | boilerplate |
| **Exposed** (JetBrains) | Kotlin-first ORM/DSL | Kotlin 친화 / 학습 쉬움 | Spring Boot 통합 약간 약함 / 작은 생태계 |
| **R2DBC + Coroutines** | 비동기 / suspend | webflux 와 짝 / non-blocking | 학습 + 운영 부담 |

---

## 2. 의사결정 트리

(Java 원본 [[../../java-spring/database/database#2 의사결정 트리|§2]] 와 동일 — Kotlin 추가 분기:)

```
Q5. Kotlin-first 도구 선호 + Spring Boot 통합 OK ?
├─ Yes → Exposed (JetBrains) 또는 JPA (kotlin-jpa plugin)
└─ No  → JPA / MyBatis (Java 와 동일)

Q6. webflux / 비동기 + suspend ?
├─ Yes → R2DBC + kotlinx-coroutines-reactor
└─ No  → JPA / JDBC (blocking)
```

### 실전 패턴 (한국 SaaS — Kotlin)

| 시나리오 | 권장 |
| --- | --- |
| 신규 e-commerce / SaaS (서버사이드) | **JPA + kotlin-jpa plugin** 기본 + 보고서 MyBatis |
| 레거시 SI 의 Kotlin 도입 | **MyBatis** + kotlin null-safety 주의 |
| OLAP / 분석 | **JOOQ Kotlin DSL** (data class 매핑) |
| 마이크로서비스 webflux | **R2DBC + Coroutines** |
| Admin 도구 / 단순 CRUD | **JdbcTemplate** 또는 **Exposed** |

→ 한국 Kotlin 진영에서도 **JPA + MyBatis 공존** 이 가장 흔함.

상세 가이드:
- [[jpa]] — Kotlin + JPA 본편 (kotlin-jpa plugin / data class 회피 / by lazy)
- [[mybatis]] — Kotlin + MyBatis 본편
- [[jpa-mybatis-coexist]] — 둘 다 사용

---

## 3. JPA vs MyBatis — 1:1 비교

(Java 원본 [[../../java-spring/database/database#3 JPA vs MyBatis|§3]] 와 동일 + Kotlin 항목 추가)

| 항목 | JPA + Kotlin | MyBatis + Kotlin |
| --- | --- | --- |
| **Entity 형태** | `open class` + `var` (data class 회피) | data class (DTO) + result map |
| **null-safety** | Hibernate 가 null 인식 약함 — `?` 명시 | mapper 가 `?` 자동 매핑 (mybatis-spring-boot-starter) |
| **plugin 필요** | `kotlin("plugin.spring")` + `kotlin("plugin.jpa")` | `kotlin("plugin.spring")` (Mapper interface proxy) |
| **DTO Projection** | `interface` + nullable getter | data class 생성자 매핑 |

→ 자세한 함정은 각 노트의 "Kotlin 한정" 절.

---

## 4. DataSource / HikariCP

ORM 무관 — DB 연결 풀은 **HikariCP** (Spring Boot 기본).

(yaml 설정은 Java 원본 [[../../java-spring/database/database#4 DataSource / HikariCP|§4]] 와 동일)

---

## 5. 마이그레이션 — Flyway (ORM 무관)

```kotlin
implementation("org.flywaydb:flyway-core")
implementation("org.flywaydb:flyway-database-postgresql")
```

(yaml / SQL 규칙은 Java 원본 §5 와 동일. 자세한 Flyway 절은 [[flyway]].)

---

## 6. 이 폴더의 노트

```
kotlin-spring/database/
├── database.md                  ← 본 hub
├── jpa.md                       ← Spring Data JPA + Kotlin
├── mybatis.md                   ← MyBatis + Kotlin
├── jpa-mybatis-coexist.md       ← 둘 다 사용 시 세팅 / 패턴
├── flyway.md                    ← Flyway 마이그레이션
├── jooq-kotlin.md               (예정)
├── exposed.md                   (예정)
└── r2dbc-coroutines.md          (예정)
```

진행:

| 노트 | 상태 |
| --- | --- |
| [[jpa]] | ✅ |
| [[mybatis]] | ✅ |
| [[jpa-mybatis-coexist]] | ✅ |
| [[flyway]] | ✅ |
| jooq-kotlin | 🟡 |
| exposed | 🟡 |
| r2dbc-coroutines | 🟡 |

---

## 7. 레시피와의 연결

(Java 원본 §7 의 표 그대로 — Kotlin 진영도 동일한 권장)

---

## 8. Kotlin 한정 운영·디버깅 체크리스트

- [ ] `kotlin("plugin.spring")` 적용 (final class proxy)
- [ ] `kotlin("plugin.jpa")` 적용 (`@Entity` no-arg 생성자)
- [ ] JPA Entity 는 `open class` + `var` (data class 회피)
- [ ] `@field:Column`, `@field:JoinColumn` use-site target 명시
- [ ] `KotlinModule` ObjectMapper 등록 (response 직렬화)
- [ ] mybatis-spring-boot-starter 의 `useColumnLabel` 설정 확인
- [ ] coroutines + JDBC 혼용 시 `Dispatchers.IO` 명시 (block 차단)

---

## 9. 관련

- [[../../java-spring/database/database|↗ Java 원본]]
- [[../api-design/api-design|↗ API 레시피 hub]]
- [[../../../../database/database|↗ 20-areas/database (DB 자체)]]
- [[../kotlin-spring|↑ Kotlin Spring]]
