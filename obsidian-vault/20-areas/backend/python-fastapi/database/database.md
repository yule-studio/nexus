---
title: "Python FastAPI — Data Access / ORM Hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-27T00:55:00+09:00
tags:
  - backend
  - python-fastapi
  - database
  - orm
  - hub
---

# Python FastAPI — Data Access / ORM Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | placeholder |
| v.2.0.0 | 2026-05-27 | engineering-agent/tech-lead | java-spring/database hub 미러링 + SQLAlchemy 2.x async |

**[[../python-fastapi|↑ FastAPI]]** · **[[../../spring/java-spring/database/database|↗ Java 원본]]** · **[[../../python-flask/database/database|↗ Flask 변형]]**

> FastAPI 진영의 표준 = **SQLAlchemy 2.x async + asyncpg + Alembic async template**.

---

## 1. Python FastAPI 의 Data Access 도구

| 도구 | 한 줄 | 강점 | 약점 |
| --- | --- | --- | --- |
| **SQLAlchemy 2.x async** | async ORM + Core | FastAPI 와 자연스러움 | async session lifecycle 학습 |
| **SQLModel** | SQLAlchemy + pydantic 통합 | FastAPI 친화 (creator: tiangolo) | 작은 생태계 |
| **Tortoise ORM** | Django ORM 스타일 async | 단순 | 작은 생태계 |
| **databases** (encode) | SQLAlchemy Core async wrapper | 가벼움 | maintenance 약화 |
| **MyBatis 대응 — `db.execute(text(...))`** | raw SQL | tuning / 복잡 쿼리 | boilerplate |

→ **표준** = SQLAlchemy 2.x async + `text()` (raw) 혼용.

---

## 2. sync vs async — FastAPI 는 async

| | Flask (sync) | FastAPI (async) |
| --- | --- | --- |
| 드라이버 | `psycopg` (sync) | **`asyncpg`** 또는 `psycopg[async]` |
| Session | `db.session` | `AsyncSession` |
| 호출 | `db.session.query(X).all()` | `await db.scalars(select(X)).all()` |
| Migration | Flask-Migrate / Alembic | **Alembic (async template)** |

---

## 3. 표준 의존성

```bash
pip install "SQLAlchemy[asyncio]>=2.0.29" asyncpg alembic
```

```python
# app/db.py
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase

from app.config import settings

engine = create_async_engine(
    settings.DATABASE_URL,                  # postgresql+asyncpg://...
    echo=False,
    pool_pre_ping=True,
    pool_size=20,
    max_overflow=10,
)

AsyncSessionLocal = async_sessionmaker(
    engine, class_=AsyncSession, expire_on_commit=False,
)


class Base(DeclarativeBase):
    pass


async def get_db():
    """FastAPI Depends — async session per request."""
    async with AsyncSessionLocal() as session:
        yield session
```

---

## 4. 추천 패키지 구조

```
app/
├── db.py                       # engine + AsyncSessionLocal + Base
├── domain/                     # SQLAlchemy 2.x async 모델
├── repositories/               # async 패턴
├── services/                   # async business logic
├── api/                        # APIRouter
└── deps.py                     # SessionDep
```

---

## 5. 마이그레이션 — Alembic async template

```bash
alembic init -t async migrations
```

(자세한 설정은 [[../getting-started#4.4 Alembic 마이그레이션|getting-started §4.4]] 참고)

```bash
alembic revision --autogenerate -m "create users"
alembic upgrade head
```

> **운영 절대 규칙**: `Base.metadata.create_all()` 운영 X. 항상 `alembic upgrade head`.

---

## 6. 이 폴더의 노트

```
database/
├── database.md                ← 본 hub
├── jpa.md                     ← SQLAlchemy 2.x async 본편
├── mybatis.md                 ← raw SQL (text() / Core) async
├── jpa-mybatis-coexist.md     ← ORM + raw 공존
└── flyway.md                  ← Alembic async (Flyway 동등)
```

진행:

| 노트 | 상태 |
| --- | --- |
| [[jpa]] | ✅ |
| [[mybatis]] | ✅ |
| [[jpa-mybatis-coexist]] | ✅ |
| [[flyway]] | ✅ |

---

## 7. FastAPI 한정 운영·디버깅 체크리스트

- [ ] `postgresql+asyncpg://` URL
- [ ] `AsyncSession` (sync Session X)
- [ ] `expire_on_commit=False` (response model 직렬화 시점에 attribute 살아있음)
- [ ] `pool_pre_ping=True`
- [ ] uvicorn `--workers N` × `pool_size` ≤ PG `max_connections`
- [ ] `alembic upgrade head` 가 컨테이너 startup (또는 별도 init container)
- [ ] N+1 회피 — `selectinload` / `joinedload` (async 호환 strategy)
- [ ] `engine.dispose()` 가 lifespan shutdown
- [ ] async endpoint 안 sync DB query 절대 X (event loop block)

---

## 8. 관련

- [[../../spring/java-spring/database/database|↗ Java 원본]]
- [[../../python-flask/database/database|↗ Flask 변형]]
- [[../api-design/api-design|↗ API 레시피 hub]]
- [[../getting-started]] — FastAPI 입문 가이드 §4 DB 셋업
- [[../python-fastapi|↑ FastAPI]]
