---
title: "Python Flask — API 구현 레시피 Hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T17:00:00+09:00
tags:
  - backend
  - python-flask
  - api-design
  - recipe
  - hub
---

# Python Flask — API 구현 레시피 Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 폴더 hub placeholder |
| v.2.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring/api-design hub 미러링 + Flask stack 규약 |

**[[../python-flask|↑ Flask]]** · **[[../../spring/java-spring/api-design/api-design|↗ Java 원본]]**

> Flask + Python 3.12 기준 실전 레시피 hub. 각 레시피는 Java 원본과 동일 구조 (전제 / 도메인 / DB / 보안 / 구현 / 테스트 / 운영) 를 Flask 패턴으로.

---

## 0. 표준 스택

| 영역 | 라이브러리 |
| --- | --- |
| Python | **3.12+** |
| Web | **Flask 3.x** |
| ORM | SQLAlchemy 2.x + Flask-SQLAlchemy / Alembic |
| Auth | Flask-JWT-Extended + argon2-cffi |
| Cache | Flask-Caching + redis-py |
| Validation | marshmallow 3.x 또는 pydantic 2.x |
| Async / Worker | Celery 5.x + redis broker |
| HTTP client | httpx 또는 requests |
| 테스트 | pytest + pytest-flask + factory_boy |
| Lint / format | ruff + black + mypy |

```python
# pyproject.toml
[project]
dependencies = [
    "flask>=3.0",
    "Flask-SQLAlchemy>=3.1",
    "Flask-Migrate>=4.0",
    "Flask-JWT-Extended>=4.6",
    "Flask-CORS>=4.0",
    "Flask-Caching>=2.1",
    "argon2-cffi>=23.1",
    "marshmallow>=3.20",
    "redis>=5.0",
    "celery[redis]>=5.3",
    "httpx>=0.27",
    "psycopg[binary]>=3.1",
]
```

---

## 1. 각 레시피의 표준 목차

(Java 원본 [[../../spring/java-spring/api-design/api-design#1 각 레시피의 표준 목차|§1]] 의 12 섹션 그대로 — §6 "구현" 만 Flask)

---

## 2. ORM 정책 — SQLAlchemy

Flask 진영의 표준은 **SQLAlchemy 2.x + Alembic**.

- **도메인 CUD** — SQLAlchemy ORM (declarative + session)
- **복잡 조회 / 보고서** — SQLAlchemy Core 또는 `session.execute(text(...))`

상세: [[../database/database]].

---

## 3. 추천 패키지 구조

```
app/
├── __init__.py                 # create_app() factory
├── config.py
├── domain/                     # framework 모름. POPO + 도메인 로직
├── application/                # service / usecase
├── repositories/               # SQLAlchemy adapter
├── api/                        # Flask blueprint
├── schemas/                    # marshmallow / pydantic
├── common/                     # envelope / exceptions / jwt / cors
└── workers/                    # Celery
```

---

## 4. 공통 원칙

(Java 원본 §4 의 OOP / DB / 보안 / 트랜잭션 / 검증 원칙 동일 — 인프라 / 알고리즘 / 정책은 언어 무관)

### 4.4 응답 / 예외 표준

[[../common/response-envelope]] 의 `CommonResponse[T]` + `BusinessException`. Endpoint 마다 try/except 없음.

---

## 5. 진행 상태 (단일 파일 — 본 PR 범위)

| 노트 | 한 줄 |
| --- | --- |
| [[cache-redis]] | Flask-Caching + redis-py |
| [[distributed-lock]] | redis-py + distributed lock recipe |
| [[rate-limiting]] | Flask-Limiter + redis backend |
| [[rbac-permissions]] | `@role_required` decorator |
| [[two-factor-auth]] | pyotp + qrcode |
| [[common-response-envelope]] | api-design 안 보조 노트 |
| [[elasticsearch-integration]] | elasticsearch-py |
| [[excel-csv-import-export]] | openpyxl + csv stdlib |
| [[feed-timeline]] | fan-out + Celery |
| [[file-upload-s3]] | boto3 + presigned URL |
| [[geo-search]] | GeoAlchemy2 (PostGIS) 또는 redis GEO |
| [[recommendation]] | event 적재 → Celery → Redis ZSET |
| [[review]] | SQLAlchemy + atomic UPDATE |
| [[webhook-send]] | Celery + httpx + HMAC |
| [[websocket-stomp]] | Flask-SocketIO |

---

## 6. 관련

- [[../python-flask|↑ Flask]]
- [[../../spring/java-spring/api-design/api-design|↗ Java 원본]]
- [[../../spring/kotlin-spring/api-design/api-design|↗ Kotlin 변형]]
- [[../common/response-envelope]]
- [[../common/security-config]]
