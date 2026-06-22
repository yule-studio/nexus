---
title: "Python / Flask (Area)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T20:30:00+09:00
tags:
  - area
  - backend
  - python-flask
---

# Python / Flask (Area)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | placeholder |
| v.2.0.0 | 2026-05-27 | engineering-agent/tech-lead | area hub — guidelines / common / api-design / database 4 영역 navigation |

**[[../backend|↑ backend]]** · **[[../python-fastapi/python-fastapi|↗ FastAPI]]** · **[[../python-django/python-django|↗ Django]]** · **[[../spring/kotlin-spring/kotlin-spring|↗ Kotlin Spring]]**

> Python 의 micro web framework — werkzeug + jinja2. 작은 / 명시적 / 확장 자유.
> **처음이면 [[guidelines/guidelines|guidelines]]** 부터 → 실전 레시피는 [[api-design/api-design|api-design]] → ORM 깊이는 [[database/database|database]].

---

## 0. 처음이라면

→ **[[guidelines/guidelines|guidelines/]]** 10 파일 입문 + 기본 세팅 가이드.

> 다른 framework 비교:
> - [[../python-fastapi/getting-started]] (단일) / 향후 guidelines/
> - [[../python-django/getting-started]] (단일) / 향후 guidelines/
> - [[../spring/kotlin-spring/guidelines/guidelines|↗ Kotlin Spring guidelines]]

---

## 1. 4 영역 (sub-area)

### 1.1 [[guidelines/guidelines|guidelines/]] — 입문 / 기본 세팅

| 노트 | 내용 |
| --- | --- |
| [[guidelines/01-installation-and-hello-world]] | Python / venv / Flask 설치 / 첫 endpoint |
| [[guidelines/02-project-structure]] | 패키지 구조 / app factory / Blueprint / .env |
| [[guidelines/03-domain-and-sqlalchemy]] | User / Memo model + SQLAlchemy 2.x + Flask-Migrate |
| [[guidelines/04-controller-and-validation]] | Blueprint route / marshmallow / 예외 처리 |
| [[guidelines/05-security-and-deployment]] | Flask-JWT-Extended / Argon2 / gunicorn + Docker |
| [[guidelines/06-swagger-openapi]] | Flask-Smorest 또는 apispec — Swagger 자동 |
| [[guidelines/07-controller-patterns]] | request / response / status / Blueprint 패턴 |
| [[guidelines/08-monitoring]] | prometheus-flask-exporter / healthz |
| [[guidelines/09-logging-mdc]] | logging / request id / JSON log |
| [[guidelines/10-devtools-hotreload]] | debug mode / `flask --reload` / Werkzeug debugger |

### 1.2 [[common/common|common/]] — 횡단 표준

| 노트 | 내용 |
| --- | --- |
| [[common/response-envelope]] | dataclass `CommonResponse` + `BusinessException` + `@app.errorhandler` |
| [[common/security-config]] | Flask-JWT-Extended / Flask-CORS / Argon2 / `@role_required` |

### 1.3 [[api-design/api-design|api-design/]] — 실전 레시피 (16 단일 파일)

회원가입 / 결제 / 캐시 / 파일 업로드 등 도메인 레시피. 자세한 표는 hub 노트.

### 1.4 [[database/database|database/]] — ORM 깊이

| 노트 | 내용 |
| --- | --- |
| [[database/database]] | ORM 선택 / Flask-SQLAlchemy |
| [[database/jpa]] | SQLAlchemy 2.x (JPA 대응) 본편 |
| [[database/mybatis]] | text() / Core (raw SQL) |
| [[database/jpa-mybatis-coexist]] | ORM + raw SQL 함께 |
| [[database/flyway]] | Alembic + Flask-Migrate |

### 1.5 [[api-docs/api-docs|api-docs/]] — API 문서화 (OpenAPI / Swagger)

### 1.6 [[pitfalls/pitfalls|pitfalls/]] — 함정 / 안티패턴

---

## 2. Flask 핵심 한 줄

- **micro framework** — Flask 자체는 routing + request/response 만. ORM / Auth / Swagger 는 별도 라이브러리.
- **app factory + Blueprint** — `create_app()` + `Blueprint("...", __name__)` 가 표준 구조.
- **WSGI** — sync. async 필요하면 Quart 또는 FastAPI.
- **`.env` + dotenv** — Config 클래스 + `python-dotenv` 가 표준.

---

## 3. Flask / FastAPI / Django / Spring 비교

| | Flask | FastAPI | Django | Spring (Kotlin) |
| --- | --- | --- | --- | --- |
| 컨셉 | **micro** | async modern | full-stack | 자동설정 |
| Type 시스템 | 선택 (hint) | **pydantic 강제** | 선택 | 강 (null-safe) |
| ORM | 별도 (SQLAlchemy) | 별도 (SQLAlchemy async) | **내장** | JPA |
| Migration | Flask-Migrate | Alembic | **내장** | Flyway |
| Auth | Flask-JWT-Extended | python-jose | Simple JWT | Spring Security |
| Swagger | Flask-Smorest | **자동** | DRF + drf-spectacular | springdoc |
| 적합 | 작은 API / prototype / 학습 | async 마이크로서비스 | full-stack admin / 한국 SI | 큰 SaaS / 엔터프라이즈 |

---

## 4. 학습 / 외부 자료

- Flask: <https://flask.palletsprojects.com/>
- Flask-SQLAlchemy: <https://flask-sqlalchemy.palletsprojects.com/>
- Flask-JWT-Extended: <https://flask-jwt-extended.readthedocs.io/>
- marshmallow: <https://marshmallow.readthedocs.io/>
- *Flask Web Development* (Miguel Grinberg)

---

## 5. 관련

- [[../backend|↑ backend]]
- [[../python-fastapi/python-fastapi|↗ FastAPI]]
- [[../python-django/python-django|↗ Django]]
- [[../spring/kotlin-spring/kotlin-spring|↗ Kotlin Spring]]
- [[../spring/java-spring/java-spring|↗ Java Spring]]
