---
title: "03. Domain Model + SQLAlchemy 2.x + Flask-Migrate"
kind: guide
status: current
created_at: 2026-05-27T12:30:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-flask, guidelines, sqlalchemy, migrate]
---

# 03. Domain Model + SQLAlchemy 2.x + Flask-Migrate

**[[guidelines|↑ Hub]]** ← [[02-project-structure]] · → 다음 [[04-controller-and-validation]]

> User / Memo model 정의 + Spring Data JPA 동등 패턴 + Flyway 동등 (Flask-Migrate).

---

## 1. PostgreSQL 준비

**Docker** 가 가장 빠름:

```bash
docker run -d \
  --name memo-db \
  -p 5432:5432 \
  -e POSTGRES_USER=memo \
  -e POSTGRES_PASSWORD=memo \
  -e POSTGRES_DB=memo \
  postgres:16
```

연결 테스트:

```bash
psql postgresql://memo:memo@localhost:5432/memo -c "SELECT 1"
```

---

## 2. SQLAlchemy 2.x — Mapped / mapped_column

```python
# app/domain/user.py
from datetime import datetime, timezone
from typing import Optional
from sqlalchemy import String, DateTime
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.extensions import db


class User(db.Model):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    email: Mapped[str] = mapped_column(String(254), unique=True, index=True)
    password_hash: Mapped[str] = mapped_column(String(255))
    name: Mapped[str] = mapped_column(String(100))
    status: Mapped[str] = mapped_column(String(30), default="ACTIVE")
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        default=lambda: datetime.now(timezone.utc),
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        default=lambda: datetime.now(timezone.utc),
        onupdate=lambda: datetime.now(timezone.utc),
    )

    memos: Mapped[list["Memo"]] = relationship(
        back_populates="owner",
        cascade="all, delete-orphan",
        lazy="select",                              # 접근 시점에 query
    )

    def __repr__(self) -> str:
        return f"<User id={self.id} email={self.email!r}>"
```

```python
# app/domain/memo.py
from datetime import datetime, timezone
from typing import TYPE_CHECKING
from sqlalchemy import String, Text, DateTime, ForeignKey
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.extensions import db

if TYPE_CHECKING:
    from .user import User


class Memo(db.Model):
    __tablename__ = "memos"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    owner_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    title: Mapped[str] = mapped_column(String(200))
    content: Mapped[str] = mapped_column(Text, default="")
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        default=lambda: datetime.now(timezone.utc),
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        default=lambda: datetime.now(timezone.utc),
        onupdate=lambda: datetime.now(timezone.utc),
    )

    owner: Mapped["User"] = relationship(back_populates="memos", lazy="joined")

    def is_owned_by(self, user_id: int) -> bool:
        return self.owner_id == user_id

    def update(self, title: str | None, content: str | None) -> None:
        if title is not None:
            self.title = title
        if content is not None:
            self.content = content
```

> **SQLAlchemy 2.x 핵심**:
> - `Mapped[X]` + `mapped_column(...)` — type hint 기반 declarative. (1.x 의 `Column(X)` 는 deprecated.)
> - `relationship(back_populates="...")` — 양방향 관계.
> - `lazy="select"` (default) / `lazy="joined"` / `lazy="selectin"` — fetch 전략.

### 2.1 lazy 옵션 비교

| lazy | 동작 |
| --- | --- |
| `"select"` (default) | 접근 시점에 별도 query → N+1 위험 |
| `"joined"` | parent 조회 시 JOIN — 1:1 / N:1 에 |
| `"selectin"` | 별도 IN query — 1:N 큰 collection 에 |
| `"dynamic"` | 접근 시 Query 객체 반환 — 추가 필터 가능 |
| `"raise"` | 접근 시 예외 — strict N+1 검출 |

---

## 3. Flask-Migrate (Alembic wrapper)

```bash
export FLASK_APP=wsgi.py

# 첫 init (한 번만)
flask db init

# 모델 변경 → migration 생성
flask db migrate -m "create users and memos"
# migrations/versions/abc1234_create_users_and_memos.py 생성

# 적용
flask db upgrade

# 롤백
flask db downgrade -1

# 현재 버전
flask db current

# 히스토리
flask db history
```

### 3.1 생성된 migration 검토

`flask db migrate` 가 만든 파일은 **항상 검토**. autogenerate 가 일부 변경 못 잡음:

- ❌ 컬럼 rename (drop + add 로 인식)
- ❌ type 변경 (data loss 위험)
- ⚠️ index 변경 일부

```python
# migrations/versions/abc1234_create_users_and_memos.py
"""create users and memos

Revision ID: abc1234
Revises:
Create Date: 2026-05-27 12:30:00.000000
"""
from alembic import op
import sqlalchemy as sa


def upgrade():
    op.create_table('users',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('email', sa.String(length=254), nullable=False),
        sa.Column('password_hash', sa.String(length=255), nullable=False),
        sa.Column('name', sa.String(length=100), nullable=False),
        sa.Column('status', sa.String(length=30), nullable=False),
        sa.Column('created_at', sa.DateTime(timezone=True), nullable=False),
        sa.Column('updated_at', sa.DateTime(timezone=True), nullable=False),
        sa.PrimaryKeyConstraint('id')
    )
    op.create_index('ix_users_email', 'users', ['email'], unique=True)

    op.create_table('memos',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('owner_id', sa.Integer(), nullable=False),
        sa.Column('title', sa.String(length=200), nullable=False),
        sa.Column('content', sa.Text(), nullable=False),
        sa.Column('created_at', sa.DateTime(timezone=True), nullable=False),
        sa.Column('updated_at', sa.DateTime(timezone=True), nullable=False),
        sa.ForeignKeyConstraint(['owner_id'], ['users.id'],),
        sa.PrimaryKeyConstraint('id')
    )
    op.create_index('ix_memos_owner_id', 'memos', ['owner_id'])


def downgrade():
    op.drop_index('ix_memos_owner_id', table_name='memos')
    op.drop_table('memos')
    op.drop_index('ix_users_email', table_name='users')
    op.drop_table('users')
```

### 3.2 운영 절대 규칙

- ✅ V<N>__*.py 한 번 commit 후 **절대 수정 X**. 다음 revision 으로 추가만.
- ✅ migration 은 PR review 대상.
- ❌ `db.create_all()` 운영에 X. **항상 `flask db upgrade`**.
- ❌ `db.drop_all()` = 모든 데이터 날아감. dev only.

---

## 4. Repository 패턴 (선택)

작은 프로젝트는 service 안에서 직접 `db.session.query(...)` 가능. 큰 프로젝트는 repository class 로 분리:

```python
# app/repositories/user_repo.py
from typing import Optional
from sqlalchemy import select

from app.domain.user import User
from app.extensions import db


class UserRepository:

    def find_by_id(self, user_id: int) -> Optional[User]:
        return db.session.get(User, user_id)

    def find_by_email(self, email: str) -> Optional[User]:
        return db.session.scalar(select(User).where(User.email == email))

    def exists_by_email(self, email: str) -> bool:
        return db.session.scalar(select(User.id).where(User.email == email)) is not None

    def save(self, user: User) -> User:
        db.session.add(user)
        db.session.commit()
        db.session.refresh(user)
        return user

    def delete(self, user: User) -> None:
        db.session.delete(user)
        db.session.commit()


user_repository = UserRepository()                  # module-level singleton
```

---

## 5. Service — business logic

```python
# app/services/memo_service.py
from typing import Optional
from sqlalchemy import select

from app.domain.memo import Memo
from app.domain.user import User
from app.extensions import db
from app.common.exceptions import NotFoundException, ForbiddenException


def create_memo(owner: User, title: str, content: str) -> Memo:
    memo = Memo(owner_id=owner.id, title=title, content=content)
    db.session.add(memo)
    db.session.commit()
    db.session.refresh(memo)
    return memo


def list_for_owner(owner_id: int, limit: int = 20, offset: int = 0) -> list[Memo]:
    stmt = (
        select(Memo)
        .where(Memo.owner_id == owner_id)
        .order_by(Memo.created_at.desc())
        .limit(limit)
        .offset(offset)
    )
    return list(db.session.scalars(stmt).all())


def get_owned(memo_id: int, owner_id: int) -> Memo:
    memo = db.session.get(Memo, memo_id)
    if memo is None:
        raise NotFoundException("메모")
    if not memo.is_owned_by(owner_id):
        raise ForbiddenException("본인의 메모만 조회 가능합니다.")
    return memo


def update_memo(memo_id: int, owner_id: int, title: str | None, content: str | None) -> Memo:
    memo = get_owned(memo_id, owner_id)
    memo.update(title, content)
    db.session.commit()
    return memo


def delete_memo(memo_id: int, owner_id: int) -> None:
    memo = get_owned(memo_id, owner_id)
    db.session.delete(memo)
    db.session.commit()
```

> **SQLAlchemy 2.x 권장**: `db.session.scalar(select(...))` 형식. 1.x 의 `User.query.filter(...)` 는 deprecated.

---

## 6. 트랜잭션 패턴

```python
# 단순 — commit 직접
db.session.add(obj)
db.session.commit()


# 명시적 — with begin (2.x)
with db.session.begin():
    db.session.add(user)
    db.session.add(memo)
    # block 끝나면 자동 commit. 예외 시 rollback.


# try/except — 수동
try:
    db.session.add(obj)
    db.session.commit()
except IntegrityError:
    db.session.rollback()              # 필수
    raise BusinessException(...)
```

> **함정**: rollback 안 하면 다음 query 에 `PendingRollbackError`. **모든 `except` 에서 rollback**.

---

## 7. N+1 회피

```python
from sqlalchemy.orm import selectinload, joinedload

# N+1 회피 — joinedload (1:1 / N:1)
stmt = select(User).options(joinedload(User.profile))

# N+1 회피 — selectinload (1:N 큰 collection)
stmt = select(User).options(selectinload(User.memos))

# 모델 정의 단 (default)
class Memo(db.Model):
    owner: Mapped["User"] = relationship(lazy="joined")        # 항상 JOIN
```

→ 자세한 N+1 패턴은 [[../database/jpa]] 참고.

---

## 8. 자주 막히는 함정

### 함정 1 — `db.create_all()` 운영
DB schema 가 모델과 자동 동기지만 마이그레이션 history 누락. 다음 변경부터 충돌. **운영 X**.

### 함정 2 — `lazy="select"` (default) + 반복 접근
N+1. **`lazy="joined"` / `lazy="selectin"`** 또는 `options(...)`.

### 함정 3 — `db.session.commit()` 누락
변경 안 저장. service 단에서 명시.

### 함정 4 — `db.session.rollback()` 누락 후 다음 query
`PendingRollbackError`. **모든 except 에서 rollback**.

### 함정 5 — Flask-SQLAlchemy 2.x vs 3.x API
3.x 권장: `db.session.scalars(select(X))`. 2.x: `X.query.filter(...)` (deprecated).

### 함정 6 — `__repr__` 누락
debug log 에 `<User object at 0x...>`. **`__repr__` 정의**.

### 함정 7 — Migration conflict
여러 dev 가 동시에 `flask db migrate` → 같은 down_revision. **PR review + `flask db merge`**.

### 함정 8 — `lateinit` / `nullable` 혼동
`Mapped[X]` 는 NOT NULL. `Mapped[X | None]` (또는 `Optional[X]`) 는 NULL 허용. DB column 과 일치.

### 함정 9 — `Mapped[list["Memo"]]` 의 string forward reference
순환 import 회피. `from __future__ import annotations` 또는 `TYPE_CHECKING`.

### 함정 10 — Celery worker 가 `db.session` 사용
flask app context 필요. **`with app.app_context():` 명시**.

---

## 9. 다음

✅ Model + Migrate 끝.

→ [[04-controller-and-validation]] — Blueprint + marshmallow + 예외 처리
