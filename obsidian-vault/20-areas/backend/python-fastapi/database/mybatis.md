---
title: "Raw SQL async (text() / Core) — FastAPI (MyBatis 대응)"
kind: knowledge
status: current
created_at: 2026-05-27T01:10:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, database, sqlalchemy, raw-sql, async]
---

# Raw SQL async (text() / Core) — FastAPI (MyBatis 대응)

**[[database|↑ DB hub]]** · **[[../../spring/java-spring/database/mybatis|↗ Java MyBatis 원본]]** · **[[../../python-flask/database/mybatis|↗ Flask 변형 (sync)]]**

> async 버전. Flask 변형의 SQL / 매핑 / dynamic SQL 패턴 동일, `await` + `AsyncSession` 만 다름.

---

## 1-9. 도구 비교 / Mapper 패턴 / 결과 매핑 / typeHandler / JPA vs MyBatis

(Flask 변형 [[../../python-flask/database/mybatis#1|§1, §4, §5, §9, §11]] 와 동일)

---

## 2. text() — async

```python
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession


async def find_user_by_email(db: AsyncSession, email: str) -> dict | None:
    sql = text("""
        SELECT id, email, name, status, created_at
        FROM users
        WHERE lower(email) = lower(:email)
    """)
    result = await db.execute(sql, {"email": email})
    row = result.mappings().first()
    return dict(row) if row else None


async def search_users(
    db: AsyncSession, email_pattern: str | None, limit: int, offset: int,
) -> list[dict]:
    where_clauses = ["status = 'ACTIVE'"]
    params: dict = {"limit": limit, "offset": offset}

    if email_pattern:
        where_clauses.append("lower(email) LIKE concat('%', lower(:pattern), '%')")
        params["pattern"] = email_pattern

    sql = text(f"""
        SELECT id, email, name, created_at
        FROM users
        WHERE {" AND ".join(where_clauses)}
        ORDER BY created_at DESC
        LIMIT :limit OFFSET :offset
    """)
    result = await db.execute(sql, params)
    return [dict(r) for r in result.mappings().all()]
```

> **async idiom**: `await db.execute(text(...), params)` — 모든 호출 앞 await.

---

## 3. SQLAlchemy Core — async

```python
from sqlalchemy import select, update, delete, and_, or_, func


async def search_users_core(
    db: AsyncSession, pattern: str | None, limit: int,
) -> list[User]:
    stmt = (
        select(User)
        .where(
            and_(
                User.status == "ACTIVE",
                or_(
                    User.email.ilike(f"%{pattern}%") if pattern else True,
                    User.name.ilike(f"%{pattern}%") if pattern else True,
                ),
            ),
        )
        .order_by(User.created_at.desc())
        .limit(limit)
    )
    result = await db.scalars(stmt)
    return list(result.all())


async def stats_with_join(db: AsyncSession) -> list[tuple]:
    stmt = (
        select(User, func.count(Memo.id).label("memo_count"))
        .join(Memo, Memo.owner_id == User.id, isouter=True)
        .group_by(User.id)
    )
    result = await db.execute(stmt)
    return result.all()
```

---

## 4. Mapper 패턴 (MyBatis 의 `@Mapper` 대응)

```python
# app/repositories/user_mapper.py
class UserMapper:
    async def find_by_id(self, db: AsyncSession, user_id: int) -> dict | None:
        sql = text("""
            SELECT id, email, name, password_hash AS passwordHash, status,
                   created_at AS createdAt, updated_at AS updatedAt
            FROM users WHERE id = :id
        """)
        row = (await db.execute(sql, {"id": user_id})).mappings().first()
        return dict(row) if row else None

    async def insert(self, db: AsyncSession, email: str, password_hash: str, name: str) -> int:
        sql = text("""
            INSERT INTO users (email, password_hash, name, status, created_at, updated_at)
            VALUES (:email, :password_hash, :name, 'ACTIVE', now(), now())
            RETURNING id
        """)
        result = await db.execute(sql, {
            "email": email, "password_hash": password_hash, "name": name,
        })
        new_id = result.scalar_one()
        await db.commit()
        return new_id
```

---

## 5. Cursor pagination (async)

```python
async def search_after(
    db: AsyncSession,
    last_created_at: datetime | None, last_id: int | None, limit: int,
) -> list[dict]:
    if last_created_at and last_id:
        cursor_clause = "AND (created_at, id) < (:lc, :lid)"
        params = {"lc": last_created_at, "lid": last_id, "limit": limit}
    else:
        cursor_clause = ""
        params = {"limit": limit}

    sql = text(f"""
        SELECT id, email, created_at
        FROM users
        WHERE status = 'ACTIVE' {cursor_clause}
        ORDER BY created_at DESC, id DESC
        LIMIT :limit
    """)
    result = await db.execute(sql, params)
    return [dict(r) for r in result.mappings().all()]
```

---

## 6. 1:N 결과 매핑

(Flask 변형 [[../../python-flask/database/mybatis#7 1:N 결과|§7]] 의 매핑 패턴 — async 만 다름)

```python
async def find_product_with_options(db: AsyncSession, product_id: int) -> dict | None:
    sql = text("""
        SELECT
            p.id AS p_id, p.name AS p_name,
            po.id AS po_id, po.name AS po_name
        FROM products p
        LEFT JOIN product_options po ON po.product_id = p.id
        WHERE p.id = :id
    """)
    rows = (await db.execute(sql, {"id": product_id})).mappings().all()
    if not rows:
        return None

    first = rows[0]
    product = {"id": first["p_id"], "name": first["p_name"], "options": []}
    for r in rows:
        if r["po_id"]:
            product["options"].append({"id": r["po_id"], "name": r["po_name"]})
    return product
```

---

## 7. 트랜잭션

```python
async def transfer_funds(db: AsyncSession, from_id: int, to_id: int, amount: int) -> None:
    try:
        await db.execute(
            text("UPDATE accounts SET balance = balance - :amt WHERE id = :id"),
            {"amt": amount, "id": from_id},
        )
        await db.execute(
            text("UPDATE accounts SET balance = balance + :amt WHERE id = :id"),
            {"amt": amount, "id": to_id},
        )
        await db.commit()
    except Exception:
        await db.rollback()
        raise
```

---

## 8. 함정 모음

(Flask 변형 [[../../python-flask/database/mybatis#10 함정 모음|§10]] + async 한정 추가)

### FastAPI/async 한정

### 함정 11 — `db.execute(sql)` 빠진 `await`
Coroutine 그대로 반환 (실행 안 됨). 모든 호출 `await`.

### 함정 12 — sync `Result` API 와 async 혼동
`await db.execute(sql)` 의 반환은 `Result` (sync API) — `.mappings()`, `.scalar()` 등은 sync. **await 는 execute 만**.

### 함정 13 — `text("...")` 안 `IN :ids` 의 expanding param
```python
from sqlalchemy import bindparam
sql = text("SELECT * FROM users WHERE id IN :ids").bindparams(
    bindparam("ids", expanding=True),
)
await db.execute(sql, {"ids": [1, 2, 3]})
```

### 함정 14 — async session 의 `Result` 가 다른 thread 에 노출
async 함수 안에서만 처리. cross-thread 사용 X.

---

## 9. 운영 체크리스트

(Flask 변형 §12 + async)

- [ ] 모든 `db.execute` 가 `await`
- [ ] parameter binding (`:x`) 강제
- [ ] async session 안에서만 result 처리

---

## 10. 관련

- [[../../spring/java-spring/database/mybatis|↗ Java MyBatis 원본]]
- [[../../python-flask/database/mybatis|↗ Flask 변형 (sync)]]
- [[jpa]] — async ORM (JPA 대응)
- [[jpa-mybatis-coexist]] — ORM + raw 함께
- [[database|↑ database hub]]
