---
title: "SQLAlchemy 2.x — Flask 본편 (JPA 대응)"
kind: knowledge
status: current
created_at: 2026-05-26T22:45:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-flask, database, sqlalchemy, orm]
---

# SQLAlchemy 2.x — Flask 본편 (JPA 대응)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring/database/jpa.md 미러링 + Flask-SQLAlchemy 2.x + Python idiom |

**[[database|↑ DB hub]]** · **[[../../spring/java-spring/database/jpa|↗ Java 원본 (JPA)]]**

> 파일 이름은 `jpa.md` 지만 내용은 **SQLAlchemy 2.x** (Python 진영의 JPA 대응). vault 전체에서 비교 용이하도록 같은 이름.
> 가장 흔한 사고는 **N+1**, **session lifecycle**, **lazy load + commit 후 detached**.

---

## 1. JPA 와 SQLAlchemy — 1:1 매핑

| JPA | SQLAlchemy |
| --- | --- |
| `@Entity` | `class X(Base): __tablename__ = "x"` |
| `EntityManager` | `Session` (`db.session`) |
| `@Transactional` | `with db.session.begin():` 또는 `db.session.commit()` |
| 영속성 컨텍스트 / 1차 캐시 | Identity map (session 안) |
| dirty checking | autoflush + `commit()` 시 변경 감지 |
| `@OneToMany(LAZY)` | `relationship(..., lazy="select")` |
| Repository (interface) | 직접 class 또는 `db.session.query(X)` |
| `findById(id)` | `db.session.get(X, id)` |
| JPQL | SQLAlchemy ORM `select()` 또는 string SQL |
| EntityGraph (N+1 회피) | `selectinload(X.children)` |

---

## 2. 표준 의존성

```bash
pip install Flask-SQLAlchemy SQLAlchemy psycopg[binary]
```

```python
# app/config.py
class Config:
    SQLALCHEMY_DATABASE_URI = "postgresql+psycopg://memo:memo@localhost:5432/memo"
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    SQLALCHEMY_ENGINE_OPTIONS = {
        "pool_size": 20,
        "max_overflow": 10,
        "pool_pre_ping": True,
        "pool_recycle": 1800,
    }
```

---

## 3. Entity — 매핑 기본

```python
# app/domain/user.py
from datetime import datetime, timezone
from sqlalchemy import String, DateTime
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.extensions import db


class User(db.Model):
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
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        default=lambda: datetime.now(timezone.utc),
        onupdate=lambda: datetime.now(timezone.utc),
    )

    memos: Mapped[list["Memo"]] = relationship(
        back_populates="owner",
        cascade="all, delete-orphan",
        lazy="select",          # default — 접근 시점에 query
    )

    def __repr__(self) -> str:
        return f"<User id={self.id} email={self.email!r}>"
```

> **SQLAlchemy 2.x `Mapped` / `mapped_column`** — type hint 기반 declarative. Java JPA 의 `@Entity` 와 비슷한 명확성.
> **`db.Model`** — Flask-SQLAlchemy 가 제공하는 declarative base.

### 3.1 ID 전략

| 전략 | SQLAlchemy |
| --- | --- |
| AUTO_INCREMENT (PG SERIAL) | `mapped_column(primary_key=True)` (default Integer) |
| UUID | `mapped_column(UUID(as_uuid=True), default=uuid.uuid4)` |
| **ULID / 직접 발급** | `mapped_column(String(26), default=ulid_gen)` 권장 |

```python
from sqlalchemy.dialects.postgresql import UUID
import uuid

class User(db.Model):
    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
```

### 3.2 enum / JSON / Embedded

```python
from enum import Enum
from sqlalchemy import Enum as SAEnum
from sqlalchemy.dialects.postgresql import JSONB


class UserStatus(str, Enum):
    ACTIVE = "ACTIVE"
    INACTIVE = "INACTIVE"
    DELETED = "DELETED"


class User(db.Model):
    status: Mapped[UserStatus] = mapped_column(
        SAEnum(UserStatus, native_enum=False, length=30),
        default=UserStatus.ACTIVE,
    )

    metadata_json: Mapped[dict] = mapped_column(JSONB, default=dict)
```

> **`native_enum=False`** — DB 의 ENUM type 안 만들고 VARCHAR 사용 (마이그레이션 / 호환 쉬움).
> **`JSONB`** — PG only. SQLite 등은 `JSON`.

### 3.3 컬렉션 / 관계

```python
class Memo(db.Model):
    __tablename__ = "memos"

    id: Mapped[int] = mapped_column(primary_key=True)
    owner_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    title: Mapped[str] = mapped_column(String(200))

    owner: Mapped["User"] = relationship(back_populates="memos", lazy="joined")
    #              ^^^^^^^^^^^^^^^^^^^^^^^^
    # lazy="joined" — JOIN 자동 (N+1 회피, 항상 같이 fetch)
    # lazy="selectin" — 별도 query (1:N 큰 collection 추천)
    # lazy="select" — default (필요 시점에 query — N+1 위험)
```

> **함정**: `lazy="select"` (default) + 반복 접근 = N+1. **`selectinload` / `joinedload`** 옵션 명시.

---

## 4. Repository — SQLAlchemy 2.x select

```python
# app/repositories/user_repo.py
from sqlalchemy import select
from sqlalchemy.orm import Session

from app.domain.user import User
from app.extensions import db


class UserRepository:

    def find_by_id(self, user_id: int) -> User | None:
        return db.session.get(User, user_id)

    def find_by_email(self, email: str) -> User | None:
        stmt = select(User).where(User.email == email)
        return db.session.scalars(stmt).first()

    def exists_by_email(self, email: str) -> bool:
        stmt = select(User.id).where(User.email == email)
        return db.session.scalars(stmt).first() is not None

    def search(self, email_pattern: str | None, limit: int = 20, offset: int = 0) -> list[User]:
        stmt = select(User)
        if email_pattern:
            stmt = stmt.where(User.email.ilike(f"%{email_pattern}%"))
        stmt = stmt.order_by(User.created_at.desc()).limit(limit).offset(offset)
        return list(db.session.scalars(stmt).all())

    def save(self, user: User) -> User:
        db.session.add(user)
        db.session.commit()
        db.session.refresh(user)
        return user

    def delete(self, user: User) -> None:
        db.session.delete(user)
        db.session.commit()

    # 비관 락 — SELECT ... FOR UPDATE
    def find_for_update(self, user_id: int) -> User | None:
        stmt = select(User).where(User.id == user_id).with_for_update()
        return db.session.scalars(stmt).first()

    # 일괄 update
    def update_status_bulk(self, ids: list[int], status: UserStatus) -> int:
        from sqlalchemy import update
        stmt = update(User).where(User.id.in_(ids)).values(status=status)
        result = db.session.execute(stmt)
        db.session.commit()
        return result.rowcount
```

> **SQLAlchemy 2.x 권장**: `db.session.scalars(select(...))` 형식. 1.x 의 `User.query.filter(...)` 는 deprecated.

### 4.1 EntityGraph 동등 — selectinload

```python
from sqlalchemy.orm import selectinload, joinedload

# N+1 회피
stmt = select(User).options(
    selectinload(User.memos),       # 1:N — 별도 query (큰 collection)
    joinedload(User.profile),       # 1:1 / N:1 — JOIN
)
users = db.session.scalars(stmt).all()

for user in users:
    print(user.memos[0].title)      # 추가 query 없음
```

---

## 5. 영속성 컨텍스트 (Session)

```python
# autoflush + commit 시 dirty checking
user = db.session.get(User, 1)
user.name = "changed"               # 객체만 변경. DB 호출 X.
db.session.commit()                  # 여기서 UPDATE 발사
```

> **`db.session`** = Flask-SQLAlchemy 가 request 마다 새로. Spring `@RequestScope` 동등.

### 5.1 트랜잭션 — autoflush + commit

```python
# Flask-SQLAlchemy default: autocommit=False, autoflush=True
# 매 query 전에 변경 사항 flush (DB 보냄, commit 은 따로)

@bp.post("/users")
def create_user():
    user = User(email="a@b.com", ...)
    db.session.add(user)
    # autoflush — query 직전에 INSERT
    db.session.commit()                          # 트랜잭션 종료
    return ok(user.id)


# 명시적 트랜잭션
with db.session.begin():
    user = User(...)
    db.session.add(user)
    other = User(...)
    db.session.add(other)
# block 끝나면 자동 commit. 예외 시 rollback.
```

### 5.2 rollback

```python
try:
    db.session.add(user)
    db.session.commit()
except IntegrityError:
    db.session.rollback()                        # 필수
    raise BusinessException(...)
```

> **함정**: rollback 안 하면 다음 query 에 `PendingRollbackError`. **모든 `except`** 에서 rollback.

---

## 6. 비관 락 / 낙관 락

### 6.1 비관 락

```python
# SELECT ... FOR UPDATE
stmt = select(ProductOption).where(ProductOption.id == option_id).with_for_update()
option = db.session.scalars(stmt).first()
option.stock -= qty
db.session.commit()
```

### 6.2 낙관 락 (수동)

SQLAlchemy 2.x 는 `@Version` 자동 없음 — 수동 구현:

```python
class Order(db.Model):
    id: Mapped[int] = mapped_column(primary_key=True)
    status: Mapped[str] = mapped_column(String(30))
    version: Mapped[int] = mapped_column(default=0)

    __mapper_args__ = {
        "version_id_col": version,        # SQLAlchemy 가 자동으로 WHERE version = X 추가
    }
```

→ `commit()` 시 row 가 다른 트랜잭션에 변경됐으면 `StaleDataError`.

---

## 7. 도메인 ↔ ORM 분리 패턴

(Java 원본 [[../../spring/java-spring/database/jpa#9 도메인 ↔ JPA 분리|§9]] 와 동일 원칙)

```python
# domain/user.py — pure domain (POPO)
from dataclasses import dataclass


@dataclass
class User:
    id: int | None
    email: str
    password_hash: str
    name: str

    def change_password(self, new_hash: str) -> None:
        self.password_hash = new_hash


# repositories/jpa/user.py — SQLAlchemy entity (infrastructure)
class UserEntity(db.Model):
    __tablename__ = "users"
    # ... (SQLAlchemy 컬럼)


# repositories/user_repo.py — adapter
class UserRepository:
    def find_by_id(self, user_id: int) -> User | None:
        entity = db.session.get(UserEntity, user_id)
        return self._to_domain(entity) if entity else None

    def save(self, user: User) -> User:
        if user.id is None:
            entity = UserEntity(email=user.email, ...)
        else:
            entity = db.session.get(UserEntity, user.id)
            entity.password_hash = user.password_hash
        db.session.add(entity)
        db.session.commit()
        return self._to_domain(entity)

    def _to_domain(self, e: UserEntity) -> User:
        return User(id=e.id, email=e.email, password_hash=e.password_hash, name=e.name)
```

→ Service / domain 은 SQLAlchemy 모름. 단 boilerplate 증가 — 작은 프로젝트는 entity = domain 통합 OK.

---

## 8. 함정 모음

(Java 원본 [[../../spring/java-spring/database/jpa#11 함정 모음|§11]] 의 함정 그대로 + Python 한정)

### 함정 1 — `db.session.commit()` 누락
변경 사항 안 저장. 어디서 commit 할지 명확히 (service 단 권장).

### 함정 2 — `db.session.rollback()` 누락
`IntegrityError` 후 `PendingRollbackError` 폭증. `try/except` 모두 rollback.

### 함정 3 — N+1
`for user in users: print(user.memos)` = N 개 추가 query. **`selectinload` / `joinedload`**.

### 함정 4 — `lazy="dynamic"` 의 query 직접 반환
`user.memos` 가 collection 아니라 `Query` 객체. `.all()`, `.count()` 직접.

### 함정 5 — `Session.expire_on_commit` (default True)
commit 후 ORM 객체의 모든 attribute 가 expired → 다음 접근 시 SELECT 다시. **dict 반환 또는 `expire_on_commit=False`**.

### Python/Flask 한정

### 함정 6 — Flask request thread 와 Celery worker thread 다른 session
Celery worker 안 `db.session` = 별도 session. **`with app.app_context():`** 안에서 사용.

### 함정 7 — `db.session` thread-local
multi-thread Flask (default) 에서 OK. 그러나 asyncio (Quart) 와 혼용 시 깨짐.

### 함정 8 — Flask-SQLAlchemy 2.x vs 3.x 호환
3.x: `db.session.query()` / `db.session.scalars()`
2.x: `User.query.filter()` (deprecated)
**3.x 권장**. 옛 튜토리얼 주의.

### 함정 9 — `Mapped[X]` 와 nullable
`Mapped[X | None]` 가 SQL NULL 허용. `Mapped[X]` 는 NOT NULL. **DB column 과 일치 강제**.

### 함정 10 — `Session.commit()` 안 `db.session.refresh(obj)` 필요한 경우
INSERT 후 server-generated ID / default 값 보려면 `refresh()`. 안 호출하면 `obj.id` = None.

---

## 9. 운영 / 디버깅 체크리스트

- [ ] `SQLALCHEMY_TRACK_MODIFICATIONS = False`
- [ ] `pool_pre_ping = True`
- [ ] `pool_recycle` < DB `wait_timeout`
- [ ] dev/staging — `SQLALCHEMY_ECHO = True` 또는 logger
- [ ] N+1 검출 — `flask-debugtoolbar` 또는 explicit query count
- [ ] Flask-Migrate + `flask db upgrade` ([[flyway]])
- [ ] Celery worker 안 `app.app_context()` 강제
- [ ] gunicorn `workers × pool_size ≤ PG max_connections`

---

## 10. 관련

- [[../../spring/java-spring/database/jpa|↗ Java 원본 (JPA)]]
- [[mybatis]] — raw SQL (text())
- [[orm-coexist|jpa-mybatis-coexist]] — ORM + raw SQL 함께
- [[flyway]] — 마이그레이션 (Alembic)
- [[database|↑ database hub]]
