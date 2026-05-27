---
title: "03. SQLAlchemy 2.x async + asyncpg + Alembic"
kind: guide
status: current
created_at: 2026-05-27T15:00:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, guidelines, sqlalchemy, asyncpg, alembic]
---

# 03. SQLAlchemy 2.x async + asyncpg + Alembic

**[[guidelines|↑ Hub]]** ← [[02-project-structure]] · → 다음 [[04-controller-and-validation]]

> SQLAlchemy 2.x async ORM + asyncpg driver + Alembic async template.

---

## 1. PostgreSQL 준비

[[../../python-flask/guidelines/03-domain-and-sqlalchemy#1 PostgreSQL 준비|Flask §1]] 와 동일 docker.

---

## 2. async engine + AsyncSession

```python
# app/db.py
from typing import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase

from .config import settings

engine = create_async_engine(
    settings.DATABASE_URL,                       # postgresql+asyncpg://...
    echo=False,                                   # True 면 모든 SQL log
    pool_pre_ping=True,                           # connection 끊김 자동 감지
    pool_size=20,
    max_overflow=10,
    pool_recycle=1800,
)

AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,                       # ⚠️ 핵심 — response 직렬화 시 attribute 살아있도록
)


class Base(DeclarativeBase):
    pass


async def get_db() -> AsyncGenerator[AsyncSession, None]:
    """FastAPI Depends — request 마다 새 session."""
    async with AsyncSessionLocal() as session:
        try:
            yield session
        except Exception:
            await session.rollback()
            raise
        # async with 가 자동 close
```

> **`expire_on_commit=False`** 가 핵심 — True (default) 면 commit 후 attribute 만료 → response_model 직렬화 시 `MissingGreenlet` 에러.

---

## 3. 모델 정의 — Mapped / mapped_column

```python
# app/domain/user.py
from datetime import datetime
from sqlalchemy import String, DateTime, func
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
        server_default=func.now(),
    )

    memos: Mapped[list["Memo"]] = relationship(
        back_populates="owner",
        cascade="all, delete-orphan",
        lazy="selectin",                          # ⚠️ async 호환 — N+1 회피
    )

    def __repr__(self) -> str:
        return f"<User id={self.id} email={self.email!r}>"
```

```python
# app/domain/memo.py
from datetime import datetime
from typing import TYPE_CHECKING
from sqlalchemy import String, Text, DateTime, ForeignKey, func
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.db import Base

if TYPE_CHECKING:
    from .user import User


class Memo(Base):
    __tablename__ = "memos"

    id: Mapped[int] = mapped_column(primary_key=True)
    owner_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    title: Mapped[str] = mapped_column(String(200))
    content: Mapped[str] = mapped_column(Text, default="")
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
    )

    owner: Mapped["User"] = relationship(back_populates="memos", lazy="joined")
```

> **async 한정 함정**: `lazy="select"` (default) 면 attribute 접근 시점에 implicit query → async session 에서 `MissingGreenlet` 에러. **`lazy="selectin"` 또는 `lazy="joined"` 강제**.

---

## 4. Repository — async select

```python
# app/repositories/user_repo.py
from typing import Sequence
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.domain.user import User


class UserRepository:

    async def find_by_id(self, db: AsyncSession, user_id: int) -> User | None:
        return await db.get(User, user_id)

    async def find_by_email(self, db: AsyncSession, email: str) -> User | None:
        return await db.scalar(select(User).where(User.email == email))

    async def exists_by_email(self, db: AsyncSession, email: str) -> bool:
        return await db.scalar(select(User.id).where(User.email == email)) is not None

    async def list_all(self, db: AsyncSession, limit: int = 20, offset: int = 0) -> Sequence[User]:
        stmt = select(User).order_by(User.created_at.desc()).limit(limit).offset(offset)
        result = await db.scalars(stmt)
        return result.all()

    async def save(self, db: AsyncSession, user: User) -> User:
        db.add(user)
        await db.commit()
        await db.refresh(user)
        return user

    async def delete(self, db: AsyncSession, user: User) -> None:
        await db.delete(user)
        await db.commit()
```

> **SQLAlchemy 2.x async 핵심**:
> - 모든 호출 `await`
> - `db.scalar(select(X))` — 단건
> - `db.scalars(select(X)).all()` — 여러 건
> - `db.execute(stmt)` — `Result` 객체 (mappings / scalars 등)

---

## 5. Service — business logic (async)

```python
# app/services/memo_service.py
from typing import Sequence
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.domain.memo import Memo
from app.errors import NotFoundException, ForbiddenException


async def list_memos(db: AsyncSession, owner_id: int, limit: int = 20, offset: int = 0) -> Sequence[Memo]:
    stmt = (
        select(Memo)
        .where(Memo.owner_id == owner_id)
        .order_by(Memo.created_at.desc())
        .limit(limit)
        .offset(offset)
    )
    return (await db.scalars(stmt)).all()


async def get_owned_memo(db: AsyncSession, memo_id: int, owner_id: int) -> Memo:
    memo = await db.get(Memo, memo_id)
    if memo is None:
        raise NotFoundException("메모")
    if memo.owner_id != owner_id:
        raise ForbiddenException("본인의 메모만 조회 가능합니다.")
    return memo


async def create_memo(db: AsyncSession, owner_id: int, title: str, content: str) -> Memo:
    memo = Memo(owner_id=owner_id, title=title, content=content)
    db.add(memo)
    await db.commit()
    await db.refresh(memo)
    return memo


async def update_memo(db: AsyncSession, memo_id: int, owner_id: int, patch: dict) -> Memo:
    memo = await get_owned_memo(db, memo_id, owner_id)
    for key, value in patch.items():
        if value is not None and hasattr(memo, key):
            setattr(memo, key, value)
    await db.commit()
    await db.refresh(memo)
    return memo


async def delete_memo(db: AsyncSession, memo_id: int, owner_id: int) -> None:
    memo = await get_owned_memo(db, memo_id, owner_id)
    await db.delete(memo)
    await db.commit()
```

---

## 6. 트랜잭션 패턴

```python
# 단순 commit
db.add(obj)
await db.commit()


# async with begin
async with db.begin():
    db.add(user)
    db.add(memo)
    # 자동 commit. 예외 시 rollback.


# 명시적 try / except
try:
    db.add(obj)
    await db.commit()
except IntegrityError:
    await db.rollback()
    raise BusinessException(...)
```

---

## 7. Alembic — async template

```bash
alembic init -t async migrations            # ⚠️ -t async 핵심
```

`migrations/env.py` 수정:

```python
# migrations/env.py
import asyncio
from logging.config import fileConfig
from sqlalchemy.ext.asyncio import async_engine_from_config
from sqlalchemy import pool
from alembic import context

from app.config import settings
from app.db import Base
from app.domain import user, memo                # noqa: F401 — 모델 인지

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


if not context.is_offline_mode():
    asyncio.run(run_migrations_online())
```

사용:

```bash
# 모델 변경 → autogenerate
alembic revision --autogenerate -m "create users and memos"

# 적용
alembic upgrade head

# 롤백
alembic downgrade -1

# 현재 버전
alembic current
```

---

## 8. N+1 회피 — selectinload / joinedload

```python
from sqlalchemy.orm import selectinload, joinedload

# joinedload — 1:1 / N:1 (JOIN)
stmt = select(Memo).options(joinedload(Memo.owner))
memo = await db.scalar(stmt)
print(memo.owner.email)                          # 추가 query 없음

# selectinload — 1:N (별도 IN query)
stmt = select(User).options(selectinload(User.memos))
users = (await db.scalars(stmt)).all()
for u in users:
    print(u.memos)                               # N+1 회피

# 모델 default (위 §3 의 lazy="selectin" / "joined")
# → 항상 같이 fetch (의도 명확할 때만)
```

---

## 9. 자주 막히는 함정

### 함정 1 — async endpoint 안 sync DB query
`db.execute(...)` (await 없이) = Coroutine 반환 (실행 X). **모든 호출 await**.

### 함정 2 — `lazy="select"` (default) + async
implicit lazy load = `MissingGreenlet`. **`lazy="selectin"`** 또는 `lazy="joined"`.

### 함정 3 — `expire_on_commit=True` (default)
commit 후 response 직렬화 시 attribute 다시 fetch → MissingGreenlet. **`expire_on_commit=False`**.

### 함정 4 — `alembic init migrations` (sync template)
async engine 호환 X → connection error. **`alembic init -t async`** 정확히.

### 함정 5 — `engine.dispose()` lifespan 누락
process 종료 시 connection 누수. lifespan shutdown 에서 호출 ([[02-project-structure#4 main.py|02 §4]]).

### 함정 6 — `Base.metadata.create_all()` 운영
마이그레이션 history 누락. **항상 `alembic upgrade head`**.

### 함정 7 — `db.refresh(obj)` 누락
INSERT 후 server-generated id / default 없음. **명시적 refresh**.

### 함정 8 — `Mapped[X]` 와 nullable
`Mapped[X]` = NOT NULL. `Mapped[X | None]` = NULL 허용. DB column 과 일치.

### 함정 9 — autogenerate 가 컬럼 rename 못 잡음
drop + add 로 인식 = data loss. **수동 수정**.

### 함정 10 — BackgroundTasks 안 main request session 공유
main session 이 close 됨 → background 에서 에러. **새 session**:
```python
async def background():
    async with AsyncSessionLocal() as bg_db:
        # ...
        await bg_db.commit()
```

---

## 10. 다음

✅ async SQLAlchemy + Alembic 끝.

→ [[04-controller-and-validation]] — APIRouter + pydantic + 예외 처리
