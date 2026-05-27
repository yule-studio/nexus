---
title: "Python + FastAPI (Area)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T20:30:00+09:00
tags:
  - area
  - backend
  - python-fastapi
---

# Python + FastAPI (Area)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | placeholder |
| v.2.0.0 | 2026-05-27 | engineering-agent/tech-lead | area hub — guidelines / common / api-design / database 4 영역 navigation |

**[[../backend|↑ backend]]** · **[[../python-flask/python-flask|↗ Flask]]** · **[[../python-django/python-django|↗ Django]]** · **[[../spring/kotlin-spring/kotlin-spring|↗ Kotlin Spring]]**

> Modern async Python web framework — type hint + pydantic + 자동 OpenAPI.
> **처음이면 [[guidelines/guidelines|guidelines]]** 부터.

---

## 0. 처음이라면

→ **[[guidelines/guidelines|guidelines/]]** 10 파일 입문 + 기본 세팅 가이드.

---

## 1. 4 영역 (sub-area)

### 1.1 [[guidelines/guidelines|guidelines/]] — 입문 / 기본 세팅

| 노트 | 내용 |
| --- | --- |
| [[guidelines/01-installation-and-hello-world]] | Python / venv / FastAPI / uvicorn / 첫 endpoint |
| [[guidelines/02-project-structure]] | 패키지 / pydantic-settings / lifespan |
| [[guidelines/03-domain-and-sqlalchemy]] | SQLAlchemy 2.x async + asyncpg + Alembic |
| [[guidelines/04-controller-and-validation]] | APIRouter / pydantic / 예외 |
| [[guidelines/05-security-and-deployment]] | python-jose / passlib / OAuth2PasswordBearer / Docker |
| [[guidelines/06-swagger-openapi]] | 자동 /docs + customize + tag / Authorize |
| [[guidelines/07-controller-patterns]] | Path / Query / Body / Depends / response_model |
| [[guidelines/08-monitoring]] | prometheus-fastapi-instrumentator / healthz |
| [[guidelines/09-logging-mdc]] | structlog + asgi-correlation-id |
| [[guidelines/10-devtools-hotreload]] | uvicorn --reload / pytest-asyncio |

### 1.2 [[common/common|common/]] — 횡단 표준

| 노트 | 내용 |
| --- | --- |
| [[common/response-envelope]] | pydantic CommonResponse + exception_handler |
| [[common/security-config]] | OAuth2PasswordBearer + Depends + Argon2 |

### 1.3 [[api-design/api-design|api-design/]] — 실전 레시피 (16 단일 파일)

### 1.4 [[database/database|database/]] — async ORM 깊이

---

## 2. FastAPI 핵심 한 줄

- **type hint 가 곧 validation + OpenAPI** — `name: str` 만 써도 자동 documentation.
- **async / await** — non-blocking, 동시 수천 요청.
- **`Depends`** — DI built-in. Spring 의 `@Autowired` 동등.
- **자동 `/docs` Swagger** — 별도 설정 X.

---

## 3. Flask / FastAPI / Django / Spring 비교

[[../python-flask/python-flask#3 비교|↑ python-flask 비교 표]] 참조.

---

## 4. 학습 자료

- FastAPI: <https://fastapi.tiangolo.com/>
- pydantic: <https://docs.pydantic.dev/>
- SQLAlchemy async: <https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html>

---

## 5. 관련

- [[../backend|↑ backend]]
- [[../python-flask/python-flask|↗ Flask]]
- [[../python-django/python-django|↗ Django]]
- [[../spring/kotlin-spring/kotlin-spring|↗ Kotlin Spring]]
