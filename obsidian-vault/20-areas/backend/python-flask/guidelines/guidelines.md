---
title: "Flask Guidelines"
kind: guide
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-27T12:00:00+09:00
tags:
  - backend
  - python-flask
  - guidelines
  - getting-started
  - hub
---

# Flask Guidelines

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | engineering-agent/tech-lead | 10 파일 입문 + 기본 세팅 hub |

**[[../python-flask|↑ Flask]]** · **[[../../spring/kotlin-spring/guidelines/guidelines|↗ Kotlin Spring guidelines]]** · **[[../../python-fastapi/getting-started|↗ FastAPI]]** · **[[../../python-django/getting-started|↗ Django]]**

> **0 부터 Flask 로 프로젝트 만들기** + 운영 가능한 수준 세팅까지. 메모 API 예시 — 회원가입 / 로그인 / JWT / CRUD / 배포.
> 사전 지식: Python 기본 (`if/for/def/class`). HTTP / JSON / DB 약간.

---

## 0. 학습 순서 (입문 — 메모 API)

| 단계 | 노트 | 내용 |
| --- | --- | --- |
| 1 | [[01-installation-and-hello-world]] | Python / venv / Flask 설치 / 첫 endpoint |
| 2 | [[02-project-structure]] | 패키지 구조 / app factory / Blueprint / .env |
| 3 | [[03-domain-and-sqlalchemy]] | User / Memo + SQLAlchemy 2.x + Flask-Migrate |
| 4 | [[04-controller-and-validation]] | route / marshmallow / 예외 / 응답 envelope |
| 5 | [[05-security-and-deployment]] | Flask-JWT-Extended / Argon2 / gunicorn + Docker |

## 0.1 기본 세팅 — 매 프로젝트마다 적용

| 단계 | 노트 | 내용 |
| --- | --- | --- |
| 6 | [[06-swagger-openapi]] | Flask-Smorest 또는 apispec — Swagger UI 자동 |
| 7 | [[07-controller-patterns]] | request / response / status code 다양한 패턴 |
| 8 | [[08-monitoring]] | prometheus-flask-exporter / healthz |
| 9 | [[09-logging-mdc]] | logging / request_id / JSON log |
| 10 | [[10-devtools-hotreload]] | debug mode / `flask --reload` / Werkzeug debugger |

---

## 0.2 무엇을 만들 건가

같은 **메모 API** — Kotlin / FastAPI / Django guideline 과 동일 spec:

```
POST /api/v1/auth/signup          # 회원가입
POST /api/v1/auth/login           # JWT 발급

GET    /api/v1/memos              # 내 메모 목록
POST   /api/v1/memos              # 메모 생성
GET    /api/v1/memos/{id}         # 단건
PATCH  /api/v1/memos/{id}         # 수정
DELETE /api/v1/memos/{id}         # 삭제
```

PostgreSQL + JWT 인증 + Docker 배포.

---

## 0.3 Flask 의 4가지 특징

1. **micro** — Flask 자체는 routing + request/response 만. 모든 게 별도 라이브러리 — 선택권 자유.
2. **sync (WSGI)** — async 가 필요하면 Quart 또는 FastAPI.
3. **decorator 기반** — `@app.get("/x")` 한 줄로 endpoint.
4. **app factory + Blueprint** — 큰 프로젝트 확장 패턴.

### Flask vs FastAPI vs Django

| | Flask | FastAPI | Django |
| --- | --- | --- | --- |
| 학습 곡선 | **가장 쉬움** | 중간 | 큼 |
| 적합 | 작은 API / prototype / 학습 | async 마이크로서비스 | full-stack admin / SI |
| 자유도 | **가장 큼** (라이브러리 선택) | 중간 | 작음 (관례 많음) |
| Spring 비유 | Spring MVC raw | Spring WebFlux | Spring Boot starter |

자세한 비교: [[../python-flask#3 Flask / FastAPI / Django / Spring 비교|↑ python-flask §3]].

---

## 1. 가이드 순서

다음 → [[01-installation-and-hello-world]]
