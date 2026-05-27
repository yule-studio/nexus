---
title: "SQLAlchemy 2.x async — FastAPI 본편 (JPA 대응)"
kind: knowledge
status: current
created_at: 2026-05-27T01:05:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, database, sqlalchemy, orm, async]
---

# SQLAlchemy 2.x async — FastAPI 본편 (JPA 대응)

**[[database|↑ DB hub]]** · **[[../../spring/java-spring/database/jpa|↗ Java JPA 원본]]** · **[[../../python-flask/database/jpa|↗ Flask 변형 (sync)]]**

> 파일 이름은 `jpa.md` 지만 내용은 **SQLAlchemy 2.x async**. Flask 의 sync 변형과 핵심 차이만.

---

## 1. JPA / SQLAlchemy sync / SQLAlchemy async — 1:1:1

| JPA | SQLAlchemy sync | SQLAlchemy async |
| --- | --- | --- |
| `@Transactional` | `db.session.commit()` | `await db.commit()` |
| `findById` | `db.session.get(X, id)` | `await db.get(X, id)` |
| Query | `db.session.scalars(select(X)).first()` | `await db.scalars(select(X)).first()` |
| `relationship LAZY` | `lazy="select"` (proxy) | `lazy="selectin"` (eager 권장) |
| `EntityGraph` | `selectinload` | `selectinload` (동일) |
| flush | `db.session.flush()` | `await db.flush()` |

→ **거의 모든 호출 앞에 `await`** + lazy="selectin" 강제.

---

## 2. 표준 의존성

```bash
pip install "SQLAlchemy[asyncio]>=2.0.29" asyncpg
```

(Flask 변형 [[../../python-flask/database/jpa#2 표준 의존성|§2]] 의 sync 와 다른 점만 — asyncpg + asyncio extra)

---

## 3. Entity — 매핑 기본

```python
# app/domain/user.py
from datetime import datetime, timezone
from sqlalchemy import String, DateTime
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.db import Base


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(254), unique=True, index=True)
    password_hash: Mapped[str] = mapped_column(String(255))
    name: Mapped[str] = mapped_column(String(100))
    status: Mapped[str] = mapped_column(String(30), default="ACTIVE")
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        default=lambda: datetime.now(timezone.utc),
    )

    memos: Mapped[list["Memo"]] = relationship(
        back_populates="owner",
        cascade="all, delete-orphan",
        lazy="selectin",                  # ⚠️ async 호환 — N+1 회피
    )
```

> **async 한정 함정**: `lazy="select"` (default) 는 attribute 접근 시점에 implicit query → async session 에서 `MissingGreenlet` 에러. **`lazy="selectin"` 강제**.

---

## 4. Repository — async select

```python
from sqlalchemy import select, update, delete
from sqlalchemy.ext.asyncio import AsyncSession


class UserRepository:
    async def find_by_id(self, db: AsyncSession, user_id: int) -> User | None:
        return await db.get(User, user_id)

    async def find_by_email(self, db: AsyncSession, email: str) -> User | None:
        stmt = select(User).where(User.email == email)
        return await db.scalar(stmt)

    async def exists_by_email(self, db: AsyncSession, email: str) -> bool:
        stmt = select(User.id).where(User.email == email)
        return await db.scalar(stmt) is not None

    async def search(
        self, db: AsyncSession,
        email_pattern: str | None, limit: int = 20, offset: int = 0,
    ) -> list[User]:
        stmt = select(User)
        if email_pattern:
            stmt = stmt.where(User.email.ilike(f"%{email_pattern}%"))
        stmt = stmt.order_by(User.created_at.desc()).limit(limit).offset(offset)
        result = await db.scalars(stmt)
        return list(result.all())

    async def save(self, db: AsyncSession, user: User) -> User:
        db.add(user)
        await db.commit()
        await db.refresh(user)
        return user

    async def delete(self, db: AsyncSession, user: User) -> None:
        await db.delete(user)
        await db.commit()

    async def find_for_update(self, db: AsyncSession, user_id: int) -> User | None:
        stmt = select(User).where(User.id == user_id).with_for_update()
        return await db.scalar(stmt)

    async def update_status_bulk(self, db: AsyncSession, ids: list[int], status: str) -> int:
        stmt = update(User).where(User.id.in_(ids)).values(status=status)
        result = await db.execute(stmt)
        await db.commit()
        return result.rowcount
```

### 4.1 EntityGraph 동등 — selectinload async

```python
from sqlalchemy.orm import selectinload, joinedload


# async with selectinload
stmt = select(User).options(
    selectinload(User.memos),               # 1:N — 별도 query
    joinedload(User.profile),               # 1:1 / N:1 — JOIN
)
result = await db.scalars(stmt)
users = list(result.unique().all())         # joinedload + 1:N 시 unique() 필수
```

---

## 5. Session lifecycle

```python
# app/db.py
async def get_db():
    async with AsyncSessionLocal() as session:
        try:
            yield session
            # 호출자에서 commit 하지 않으면 자동 rollback (이 패턴은 X)
        except Exception:
            await session.rollback()
            raise
        # async with 가 자동 close
```

> **`expire_on_commit=False`** 가 핵심 (db.py 의 sessionmaker). True (default) 면 commit 후 attribute 만료 → response_model 직렬화 시 `MissingGreenlet`.

---

## 6. 트랜잭션

```python
async def transfer_funds(db: AsyncSession, from_id: int, to_id: int, amount: int) -> None:
    try:
        from_account = await db.get(Account, from_id, with_for_update=True)
        to_account = await db.get(Account, to_id, with_for_update=True)
        from_account.balance -= amount
        to_account.balance += amount
        await db.commit()
    except Exception:
        await db.rollback()
        raise


# 또는 명시적 transaction
async with db.begin():
    user = User(...)
    db.add(user)
    # block 끝나면 자동 commit. 예외 시 rollback.
```

---

## 7. async 한정 함정

### 함정 1 — async endpoint 안 sync DB query
`db.session.query(...)` (sync) 호출 = event loop block. **`await db.execute(select(...))`** 만.

### 함정 2 — `lazy="select"` + async
implicit lazy load 가 `MissingGreenlet` 에러. **`lazy="selectin"`** 또는 `selectinload(...)` 명시.

### 함정 3 — `expire_on_commit=True` (default)
commit 후 response model 직렬화 시 attribute 다시 fetch → `MissingGreenlet`. **`expire_on_commit=False`**.

### 함정 4 — `await db.refresh(obj)` 안 호출
INSERT 후 server-generated id / default 없음. INSERT 후 즉시 attribute 사용 시 refresh.

### 함정 5 — sync code in async generator
`async def get_db(): yield session` 안에서 sync `session.execute` 호출 = block. async 만.

### 함정 6 — BackgroundTasks 의 session 공유
main request 의 session 이 background 안에서 close 됨. **새 session**:
```python
async def background_work():
    async with AsyncSessionLocal() as db:
        # ...
        await db.commit()
```

### 함정 7 — `joinedload` + 1:N 결과
중복 row 생김 → `result.unique().all()` 필수.

### 함정 8 — Alembic + SQLAlchemy 2.x 의 type annotation
`Mapped[X]` vs `Column(X)` 혼용 시 autogenerate 가 일부 type 못 잡음. 한 스타일로 통일.

---

## 8. 운영 / 디버깅 체크리스트

- [ ] `postgresql+asyncpg://` URL
- [ ] `expire_on_commit=False`
- [ ] `lazy="selectin"` 강제 (`lazy="select"` X)
- [ ] async endpoint 안 sync DB query 절대 X
- [ ] `await db.commit()` / `await db.rollback()` 항상 await
- [ ] BackgroundTasks 의 새 session
- [ ] uvicorn workers × pool_size ≤ PG max_connections
- [ ] `engine.dispose()` lifespan shutdown

---

## 9. 관련

- [[../../spring/java-spring/database/jpa|↗ Java JPA 원본]]
- [[../../python-flask/database/jpa|↗ Flask 변형 (sync)]]
- [[mybatis]] — raw SQL async
- [[jpa-mybatis-coexist]]
- [[flyway]] — Alembic async
- [[database|↑ database hub]]
