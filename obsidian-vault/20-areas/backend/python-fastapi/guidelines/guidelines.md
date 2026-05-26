---
title: "FastAPI Guidelines"
kind: guide
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-27T14:30:00+09:00
tags:
  - backend
  - python-fastapi
  - guidelines
  - getting-started
  - hub
---

# FastAPI Guidelines

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | engineering-agent/tech-lead | 10 파일 입문 + 기본 세팅 hub |

**[[../python-fastapi|↑ FastAPI]]** · **[[../../python-flask/guidelines/guidelines|↗ Flask]]** · **[[../../python-django/getting-started|↗ Django]]** · **[[../../spring/kotlin-spring/guidelines/guidelines|↗ Kotlin Spring]]**

> **0 부터 FastAPI 로 프로젝트 만들기** + 운영 수준 세팅. 메모 API 예시 — 회원가입 / 로그인 / JWT / CRUD / 배포.

---

## 0. 학습 순서 (입문 — 메모 API)

| 단계 | 노트 | 내용 |
| --- | --- | --- |
| 1 | [[01-installation-and-hello-world]] | Python / venv / FastAPI / uvicorn / 첫 endpoint + 자동 /docs |
| 2 | [[02-project-structure]] | 패키지 / pydantic-settings / lifespan / Depends 기본 |
| 3 | [[03-domain-and-sqlalchemy]] | SQLAlchemy 2.x async + asyncpg + Alembic |
| 4 | [[04-controller-and-validation]] | APIRouter / pydantic / async / 예외 |
| 5 | [[05-security-and-deployment]] | python-jose JWT + passlib Argon2 + OAuth2PasswordBearer + Docker |

## 0.1 기본 세팅

| 단계 | 노트 | 내용 |
| --- | --- | --- |
| 6 | [[06-swagger-openapi]] | 자동 /docs + tags / Authorize / customize |
| 7 | [[07-controller-patterns]] | Path / Query / Body / Depends / response_model / status_code |
| 8 | [[08-monitoring]] | prometheus-fastapi-instrumentator / healthz |
| 9 | [[09-logging-mdc]] | structlog + asgi-correlation-id |
| 10 | [[10-devtools-hotreload]] | uvicorn --reload / fastapi dev / pytest-asyncio |

---

## 0.2 무엇을 만들 건가

같은 **메모 API** — Kotlin Spring / Flask / Django 와 동등 spec:

```
POST /api/v1/auth/signup
POST /api/v1/auth/login

GET / POST / GET{id} / PATCH{id} / DELETE{id} /api/v1/memos
```

PostgreSQL (asyncpg) + JWT + Docker.

---

## 0.3 FastAPI 의 4가지 특징

1. **type hint 가 곧 validation** — `name: str` 만 써도 422 검증 + OpenAPI schema.
2. **async / await** — non-blocking, event loop, 동시 수천 요청 처리.
3. **`Depends` DI** — Spring `@Autowired` 동등, type-safe.
4. **자동 OpenAPI** — `/docs` (Swagger) + `/redoc` 별도 설정 X.

### FastAPI vs Flask vs Django

| | FastAPI | Flask | Django |
| --- | --- | --- | --- |
| 컨셉 | **modern async** | micro sync | full-stack |
| Type | **pydantic 강제** | 선택 | 선택 |
| Swagger | **자동** | 별도 (smorest) | DRF + drf-spectacular |
| async DB | **자연스러움** | 어색 (Quart) | 옵션 (Django 4.x+) |
| 적합 | async 마이크로서비스 / 신규 | 작은 API / 학습 | full-stack admin / SI |

자세한 비교: [[../python-fastapi#3 비교|↑ python-fastapi §3]].

---

## 1. 가이드 순서

다음 → [[01-installation-and-hello-world]]
