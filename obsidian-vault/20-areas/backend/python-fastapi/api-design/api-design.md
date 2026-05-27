---
title: "Python FastAPI — API 구현 레시피 Hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T23:30:00+09:00
tags:
  - backend
  - python-fastapi
  - api-design
  - recipe
  - hub
---

# Python FastAPI — API 구현 레시피 Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | placeholder |
| v.2.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring/api-design hub 미러링 + FastAPI stack |

**[[../python-fastapi|↑ FastAPI]]** · **[[../../spring/java-spring/api-design/api-design|↗ Java 원본]]** · **[[../../python-flask/api-design/api-design|↗ Flask 변형]]**

> FastAPI + Python 3.12 기준 실전 레시피 hub. 같은 외부 계약 (JSON envelope) + async + pydantic + Depends.

---

## 0. 표준 스택

| 영역 | 라이브러리 |
| --- | --- |
| Python | **3.12+** |
| Web | **FastAPI 0.110+** |
| ASGI | uvicorn (운영은 gunicorn + UvicornWorker) |
| ORM | **SQLAlchemy 2.x async** + asyncpg + Alembic |
| Auth | python-jose + passlib[argon2] |
| Cache | aioredis 또는 redis-py async (`redis.asyncio`) |
| Validation | **pydantic 2.x** (FastAPI 기본) |
| Async / Worker | Celery 또는 FastAPI BackgroundTasks (간단) |
| HTTP client | **httpx async** |
| 테스트 | pytest-asyncio + httpx ASGITransport |

```python
# pyproject.toml
[project]
dependencies = [
    "fastapi[standard]>=0.110",
    "uvicorn[standard]>=0.27",
    "SQLAlchemy[asyncio]>=2.0.29",
    "asyncpg>=0.29",
    "alembic>=1.13",
    "python-jose[cryptography]>=3.3",
    "passlib[argon2]>=1.7",
    "pydantic>=2.6",
    "pydantic-settings>=2.2",
    "redis>=5.0",
    "httpx>=0.27",
    "celery[redis]>=5.3",
]
```

---

## 1. 각 레시피의 표준 목차

(Java 원본 [[../../spring/java-spring/api-design/api-design#1 각 레시피의 표준 목차|§1]] 의 12 섹션. §6 "구현" 은 FastAPI 패턴 — async + Depends + pydantic)

---

## 2. ORM 정책 — SQLAlchemy 2.x async

FastAPI 진영의 표준 = **SQLAlchemy 2.x async + asyncpg + Alembic async template**.

- **도메인 CUD** — `AsyncSession` (`select / db.add / await db.commit`)
- **복잡 조회 / 보고서** — `db.execute(text(...))` async

상세: [[../database/database]] (예정).

---

## 3. 추천 패키지 구조

```
app/
├── main.py                     # FastAPI() + include_router
├── config.py                   # pydantic-settings
├── deps.py                     # SessionDep / CurrentUserDep / Annotated
├── db.py                       # async engine + AsyncSessionLocal
├── domain/                     # SQLAlchemy 모델
├── schemas/                    # pydantic request/response
├── api/                        # APIRouter
├── services/                   # business logic (async)
├── errors.py                   # BusinessException + exception_handler
├── middlewares.py              # CORS / ApiLog
└── workers/                    # Celery 또는 BackgroundTasks
```

---

## 4. 공통 원칙

(Java 원본 §4 의 OOP / DB / 보안 / 트랜잭션 / 검증 원칙 동일 — 인프라 / 알고리즘 / 정책은 언어 무관)

### 4.4 응답 / 예외 표준

[[../common/response-envelope]] 의 `CommonResponse[T]` + `BusinessException`. 모든 endpoint 가 `response_model=CommonResponse[X]` 명시 — OpenAPI schema 자동.

### 4.5 비동기 우선

- DB 호출 = `await db.execute(...)`
- 외부 HTTP = `await client.get(...)` (httpx)
- sync 호출 (CPU heavy / blocking) = `run_in_executor` 또는 별도 Celery

---

## 5. 진행 상태 (단일 파일 — 본 PR 범위)

| 노트 | 한 줄 |
| --- | --- |
| [[cache-redis]] | redis.asyncio + async stampede 방어 |
| [[distributed-lock]] | redis.asyncio.lock + async context manager |
| [[rate-limiting]] | slowapi (FastAPI 친화) + redis backend |
| [[rbac-permissions]] | Depends(require_role(...)) factory |
| [[two-factor-auth]] | pyotp + AsyncSession |
| [[common-response-envelope]] | pydantic CommonResponse + RequestId middleware |
| [[elasticsearch-integration]] | elasticsearch-py async + outbox worker |
| [[excel-csv-import-export]] | openpyxl + FastAPI StreamingResponse |
| [[feed-timeline]] | async select + Celery fan-out |
| [[file-upload-s3]] | aioboto3 presigned + BackgroundTasks |
| [[geo-search]] | GeoAlchemy2 async + redis async geosearch |
| [[recommendation]] | confluent_kafka aio + redis ZSET |
| [[review]] | async select + IntegrityError + atomic UPDATE |
| [[webhook-send]] | httpx async + Celery (또는 BackgroundTasks) |
| [[websocket-stomp]] | FastAPI WebSocket native + Depends |

---

## 6. 관련

- [[../python-fastapi|↑ FastAPI]]
- [[../../spring/java-spring/api-design/api-design|↗ Java 원본]]
- [[../../python-flask/api-design/api-design|↗ Flask 변형]]
- [[../common/response-envelope]]
- [[../common/security-config]]
- [[../getting-started]] — FastAPI 입문 가이드
