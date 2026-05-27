---
title: "Kotlin + Spring (Area)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
tags:
  - area
  - backend
  - kotlin-spring
---

# Kotlin + Spring (Area)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 placeholder |
| v.2.0.0 | 2026-05-27 | engineering-agent/tech-lead | area hub — guidelines / common / api-design / database 4 영역 navigation |

**[[../backend|↑ backend]]** · **[[../java-spring/java-spring|↗ Java Spring]]**

> Spring Boot 3 + Kotlin 기준 vault area. **처음이면 [[guidelines/guidelines|guidelines]] 부터** → 실전 레시피는 [[api-design/api-design|api-design]] → ORM 깊이는 [[database/database|database]].

---

## 0. 처음이라면

→ **[[guidelines/guidelines|guidelines/]]** 5+ 파일 입문 가이드로. 설치 / 첫 프로젝트 / Domain / Controller / Security / Swagger / Logging / Actuator 까지 step-by-step.

> 이미 다른 Python framework 안다면 비교:
> - [[../../python-flask/getting-started|↗ Flask]]
> - [[../../python-fastapi/getting-started|↗ FastAPI]]
> - [[../../python-django/getting-started|↗ Django]]

---

## 1. 4 영역 (sub-area)

본 area 는 4 개 sub-area 로 구성. 학습 / 작업 단계에 맞게 진입.

### 1.1 [[guidelines/guidelines|guidelines/]] — 입문 / 기본 세팅

처음 시작 + 매번 새 프로젝트 셋업 시 참고.

| 노트 | 내용 |
| --- | --- |
| [[guidelines/01-installation-and-hello-world]] | JDK / IntelliJ / Spring Initializr / 첫 endpoint |
| [[guidelines/02-project-structure]] | 패키지 구조 / application.yml / @ConfigurationProperties |
| [[guidelines/03-domain-and-jpa]] | Entity (open class) / Repository / Flyway |
| [[guidelines/04-controller-and-validation]] | @RestController / DTO / Bean Validation / 예외 |
| [[guidelines/05-security-and-deployment]] | Spring Security / JWT / Argon2 / Docker |
| [[guidelines/06-swagger-openapi]] | Swagger UI / springdoc-openapi 통합 (작성 예정) |
| [[guidelines/07-controller-patterns]] | request 매핑 / response 형식 / status code 패턴 (작성 예정) |
| [[guidelines/08-actuator-monitoring]] | health / metrics / Prometheus (작성 예정) |
| [[guidelines/09-logging-mdc]] | Logback / MDC / X-Request-Id (작성 예정) |
| [[guidelines/10-devtools-hotreload]] | DevTools / hot reload / live reload (작성 예정) |

### 1.2 [[common/response-envelope|common/]] — 횡단 표준

모든 프로젝트에 공통으로 적용되는 envelope / security 표준.

| 노트 | 내용 |
| --- | --- |
| [[common/response-envelope]] | `CommonResponse<T>` + `BusinessException` + `ApiExceptionHandler` |
| [[common/security-config]] | SecurityConfig DSL / JWT filter / CORS / Argon2 |

### 1.3 [[api-design/api-design|api-design/]] — 실전 레시피 (16 단일 파일)

회원가입 / 결제 / 캐시 / 파일 업로드 / 알림 / 검색 등 도메인 레시피.

| 카테고리 | 노트 |
| --- | --- |
| 횡단 | [[api-design/common-response-envelope]] · [[api-design/rate-limiting]] |
| 인증 | [[api-design/rbac-permissions]] · [[api-design/two-factor-auth]] |
| 캐시 / 락 | [[api-design/cache-redis]] · [[api-design/distributed-lock]] |
| 파일 | [[api-design/file-upload-s3]] · [[api-design/excel-csv-import-export]] |
| 실시간 / 외부 | [[api-design/websocket-stomp]] · [[api-design/webhook-send]] |
| 검색 / 위치 | [[api-design/elasticsearch-integration]] · [[api-design/geo-search]] |
| 도메인 UX | [[api-design/feed-timeline]] · [[api-design/recommendation]] · [[api-design/review]] |

### 1.4 [[database/database|database/]] — ORM 깊이

JPA / MyBatis / 공존 패턴.

| 노트 | 내용 |
| --- | --- |
| [[database/database]] | ORM 선택 트리 / Hikari / 패턴 |
| [[database/jpa]] | Spring Data JPA / Hibernate 본편 (open class entity 함정 8건) |
| [[database/mybatis]] | MyBatis 본편 (data class DTO + Kotlin 함정 5건) |
| [[database/jpa-mybatis-coexist]] | JPA + MyBatis 같이 (em.flush 함정) |
| [[database/flyway]] | Flyway 마이그레이션 + Kotlin Gradle DSL |

---

## 2. Kotlin + Spring 핵심 한 줄

- **`kotlin("plugin.spring")` + `kotlin("plugin.jpa")` 필수** — 자동 `open` / no-arg 생성자.
- **JPA Entity 는 일반 `class` + `var`** — `data class` 절대 X (lazy collection 함정).
- **Spring Security DSL 은 Kotlin invoke** — `import org.springframework.security.config.annotation.web.invoke`.
- **`@Value("\${...}")` 의 `$` escape** — Kotlin string template 과 충돌.
- **`@field:Email` target 명시** — data class 파라미터에 validation 어노테이션.
- **`Optional<X>` 대신 nullable `X?`** — Spring Data 3.0+ 지원.

자세한 함정 모음은 각 sub-area 노트.

---

## 3. 같은 vault 의 다른 area 와 비교

| | Java + Spring | **Kotlin + Spring** | Flask | FastAPI | Django |
| --- | --- | --- | --- | --- | --- |
| Type | Java | **Kotlin** (null-safe) | Python | Python (pydantic) | Python |
| 비동기 | (WebFlux) | (WebFlux + coroutines) | sync | **async** | sync |
| Boilerplate | 많음 | **짧음** | 작음 | 작음 | 큼 (관례) |
| Migration | Flyway | **Flyway** | Alembic | Alembic | makemigrations |
| 적합 | 큰 SaaS / SI | **신규 Spring 프로젝트 권장** | 작은 API | async 마이크로서비스 | full-stack admin |

자세한 비교: [[guidelines/guidelines#0.3 다른 가이드와의 비교|guidelines §0.3]].

---

## 4. 학습 / 외부 자료

### 공식
- Spring Boot: <https://spring.io/projects/spring-boot>
- Spring Security: <https://docs.spring.io/spring-security/reference/>
- Spring Framework Kotlin support: <https://docs.spring.io/spring-framework/reference/languages/kotlin.html>
- Kotlin 공식: <https://kotlinlang.org/>
- jjwt: <https://github.com/jwtk/jjwt>

### 책
- *Spring in Action* (Craig Walls) — Spring 기본
- *Kotlin in Action* (Dmitry Jemerov, Svetlana Isakova) — Kotlin 문법 / idiom
- *Spring Security in Action* — Spring Security 깊이

### 강의
- Inflearn / fastcampus 의 Spring Boot / Kotlin 강의

---

## 5. 관련

- [[../backend|↑ backend]]
- [[../java-spring/java-spring|↗ Java Spring]]
- [[../../python-flask/python-flask|↗ Python Flask]]
- [[../../python-fastapi/python-fastapi|↗ Python FastAPI]]
- [[../../python-django/python-django|↗ Python Django]]
