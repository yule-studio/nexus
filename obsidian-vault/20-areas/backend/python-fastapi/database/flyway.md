---
title: "Alembic (async template) — FastAPI DB 마이그레이션"
kind: knowledge
status: current
created_at: 2026-05-27T01:00:00+09:00
agent: engineering-agent/tech-lead
tags: [knowledge, python-fastapi, alembic, migration, database]
home_hub: database
---

# Alembic (async template) — FastAPI DB 마이그레이션

**[[database|↑ DB hub]]** · **[[../../spring/java-spring/database/flyway|↗ Java Flyway]]** · **[[../../python-flask/database/flyway|↗ Flask 변형]]**

> 노트 이름은 vault 통일 위해 `flyway` 지만, FastAPI 에선 **Alembic async template**. Flask 와 다르게 `flask db` CLI 가 아니라 `alembic` 직접.

---

## 1. 왜 쓰는가

(Java 원본 [[../../spring/java-spring/database/flyway#1|§1]] 와 동일)

---

## 2. 의존성 + 초기화

```bash
pip install alembic

# async template 으로 init (중요)
alembic init -t async migrations
```

```
migrations/
├── env.py                       # async engine 설정
├── script.py.mako
└── versions/
```

---

## 3. env.py 설정

```python
# migrations/env.py
import asyncio
from logging.config import fileConfig
from sqlalchemy.ext.asyncio import async_engine_from_config
from sqlalchemy import pool
from alembic import context

from app.config import settings
from app.db import Base
from app.domain import user, memo                       # noqa: F401 — 모델 인지

config = context.config
config.set_main_option("sqlalchemy.url", settings.DATABASE_URL)
target_metadata = Base.metadata


def do_run_migrations(connection):
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


async def run_migrations_online() -> None:
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()


if context.is_offline_mode():
    pass  # offline mode skip
else:
    asyncio.run(run_migrations_online())
```

> **핵심**: `async_engine_from_config` + `connection.run_sync(do_run_migrations)` — async engine 으로 sync migration 실행 (alembic 자체는 sync).

---

## 4. 사용

```bash
# 모델 변경 → autogenerate
alembic revision --autogenerate -m "create users"

# 적용
alembic upgrade head

# 롤백
alembic downgrade -1

# 현재 버전
alembic current

# 히스토리
alembic history --verbose
```

---

## 5. Dockerfile 의 startup

```dockerfile
CMD ["sh", "-c", "alembic upgrade head && uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4"]
```

또는 별도 init container (Kubernetes):

```yaml
initContainers:
  - name: migrate
    image: app:latest
    command: ["alembic", "upgrade", "head"]
```

---

## 6. 함정 모음

(Java 원본 [[../../spring/java-spring/database/flyway#5 흔한 pitfall|§5]] + Flask 변형 함정)

### FastAPI 한정

### 함정 1 — `alembic init migrations` (sync template) 사용
async engine 과 호환 안 됨 → connection error. **`alembic init -t async`** 정확히.

### 함정 2 — env.py 의 `await connectable.dispose()` 누락
process 종료 시 connection 누수 경고. lifespan / 명시적 dispose.

### 함정 3 — autogenerate 가 SQLAlchemy 2.x `Mapped` 인식
대부분 OK 지만 일부 패턴 (예: `mapped_column(server_default=...)`) 못 잡음. 수동 수정.

### 함정 4 — Flask-Migrate 의 `flask db` CLI 와 헷갈림
FastAPI 는 `flask` 없음. **`alembic` 직접**.

### 함정 5 — Multi-instance startup 동시 migrate
Alembic 의 advisory lock 자동 — 한 instance 만, 나머지 대waiting. timeout 위험.
**CI job 하나에서만 migrate, 앱 startup 은 `alembic check`** 만.

---

## 7. 관련

- [[database]]
- [[jpa]] — SQLAlchemy async 본편
- [[../../spring/java-spring/database/flyway|↗ Java Flyway]]
- [[../../python-flask/database/flyway|↗ Flask 변형]]
- [[../getting-started#4.4 Alembic 마이그레이션|getting-started §4.4]]
