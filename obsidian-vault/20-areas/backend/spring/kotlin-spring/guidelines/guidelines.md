---
title: "Kotlin + Spring Boot Guidelines"
kind: guide
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-27T10:00:00+09:00
tags:
  - backend
  - kotlin-spring
  - guidelines
  - getting-started
  - hub
---

# Kotlin + Spring Boot Guidelines

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | engineering-agent/tech-lead | 5 파일 입문 가이드 hub |

**[[../kotlin-spring|↑ Kotlin Spring]]** · **[[../../../python-flask/getting-started|↗ Flask 가이드]]** · **[[../../../python-fastapi/getting-started|↗ FastAPI 가이드]]** · **[[../../../python-django/getting-started|↗ Django 가이드]]**

> **0 부터 Kotlin + Spring Boot 로 프로젝트 만들기**. 다른 framework 가이드와 같은 **메모 API** 프로젝트 — 끝까지 따라하면 회원가입 / 로그인 / JWT / CRUD / 배포까지.
> 사전 지식: 프로그래밍 기초 (Java / Python 어느 한쪽 정도). Spring 처음.

---

## 0. 학습 순서 (입문 — 메모 API 완성)

| 단계 | 노트 | 무엇을 배우나 |
| --- | --- | --- |
| 1 | [[01-installation-and-hello-world]] | JDK / Kotlin / IDE / Spring Initializr / 첫 실행 |
| 2 | [[02-project-structure]] | 패키지 구조 / build.gradle.kts / application.yml |
| 3 | [[03-domain-and-jpa]] | User / Memo entity (open class) + Repository + Flyway |
| 4 | [[04-controller-and-validation]] | @RestController + DTO + Bean Validation + 예외 처리 |
| 5 | [[05-security-and-deployment]] | Spring Security + JWT + Docker + 운영 체크리스트 |

## 0.1 기본 세팅 — 매 프로젝트마다 적용

01-05 따라 만든 뒤, 운영 가능한 수준으로 끌어올리는 세팅들.

| 단계 | 노트 | 무엇을 배우나 |
| --- | --- | --- |
| 6 | [[06-swagger-openapi]] | Swagger UI / springdoc-openapi 통합 — API 문서 자동 |
| 7 | [[07-controller-patterns]] | request / response / status / @PathVariable 등 패턴 정리 |
| 8 | [[08-actuator-monitoring]] | Spring Boot Actuator — health / metrics / Prometheus |
| 9 | [[09-logging-mdc]] | Logback / MDC / X-Request-Id 추적 / JSON log |
| 10 | [[10-devtools-hotreload]] | Spring DevTools / hot reload / Gradle continuous build |

---

## 0.1 무엇을 만들 건가

같은 **메모 API** — Flask / FastAPI / Django 가이드와 동일 spec:

```
POST /api/v1/auth/signup              # 회원가입
POST /api/v1/auth/login               # JWT 발급

GET    /api/v1/memos                  # 내 메모 목록
POST   /api/v1/memos                  # 메모 생성
GET    /api/v1/memos/{id}             # 단건
PATCH  /api/v1/memos/{id}             # 수정
DELETE /api/v1/memos/{id}             # 삭제
```

User 가 본인 메모만 CRUD. PostgreSQL + JWT 인증 + Docker 배포.

---

## 0.2 Kotlin + Spring 의 4가지 핵심

1. **Spring Boot auto-configuration** — 의존성만 추가하면 알아서 설정 (Java 진영 처음 = JPA / Web / Security 설정 폭주에 비교 시 천국).
2. **Kotlin null-safety** — `String?` vs `String` — NPE 가 컴파일 단계에서 차단.
3. **`kotlin("plugin.spring")` + `plugin.jpa`** — Kotlin 의 final default 클래스를 자동 `open` 으로 컴파일 → Spring proxy / JPA lazy 호환.
4. **data class + sealed class** — DTO, Value Object, 도메인 이벤트가 Java 보다 짧고 type-safe.

### Java vs Kotlin (Spring 진영) — 한 줄

| | Java + Spring | Kotlin + Spring |
| --- | --- | --- |
| Boilerplate | 많음 (getter/setter/Lombok) | 짧음 (data class) |
| Null 안전성 | Optional / @Nullable | **컴파일러 강제** |
| 람다 | functional interface | first-class function |
| 학습 곡선 | 중간 | 중간 (Java 알면 빠름) |
| 생산성 | OK | **더 짧고 정확** |

→ 신규 Spring 프로젝트면 Kotlin 권장. 단 Java 진영 호환 라이브러리 호출 시 nullable 처리 주의.

---

## 0.3 다른 가이드와의 비교

| | Spring Boot | Flask | FastAPI | Django |
| --- | --- | --- | --- | --- |
| 언어 | **Kotlin / Java** | Python | Python | Python |
| 컨셉 | full-stack + auto-config | micro | async modern | full-stack + admin |
| Type 시스템 | 강 type (Kotlin null-safety) | dynamic (선택 hint) | dynamic + **pydantic** | dynamic |
| DI | **constructor + @Autowired** | manual | `Depends` | manual / settings |
| ORM | **JPA / Hibernate** | SQLAlchemy | SQLAlchemy async | Django ORM (내장) |
| Migration | **Flyway** | Alembic | Alembic | makemigrations (내장) |
| Auth | **Spring Security + jjwt** | Flask-JWT-Extended | python-jose | Simple JWT |
| 학습 | 가장 큼 (관례 많음) | 작음 | 중간 | 큼 |
| 적합 | 대형 SaaS / 한국 SI / 엔터프라이즈 | 작은 API / prototype | async 마이크로서비스 | full-stack admin 필요 |

---

## 0.4 vault 의 다른 노트와의 관계

본 guidelines/ 는 **"처음 만들기"** 까지. 그 다음:

- 실전 레시피 (rate limit / file upload / webhook 등) → [[../api-design/api-design|api-design hub]]
- ORM 깊이 (N+1 / JPA 트랜잭션) → [[../database/database|database hub]]
- 공통 envelope / Security 표준 → [[../common/response-envelope]] / [[../common/security-config]]

---

## 1. 사전 지식 가정

- 프로그래밍 기본 (`if/for/class/function`).
- HTTP / JSON / REST API 개념 약간.
- 터미널 명령어 (`cd`, `ls`, `curl`) 약간.
- Java / Kotlin 둘 다 처음 OK — 코드는 모두 Kotlin, 필요 시 Java 비교.

본 가이드는 **Linux / macOS 기준**. Windows 는 WSL2 권장.

---

## 2. 가이드 순서

다음 노트로 → [[01-installation-and-hello-world]]
