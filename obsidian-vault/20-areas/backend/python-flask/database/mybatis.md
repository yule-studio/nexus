---
title: "Raw SQL (text() / Core) — Flask (MyBatis 대응)"
kind: knowledge
status: current
created_at: 2026-05-26T22:55:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-flask, database, sqlalchemy, raw-sql, text]
---

# Raw SQL (text() / Core) — Flask (MyBatis 대응)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring/database/mybatis.md 미러링 + SQLAlchemy text() + Core |

**[[database|↑ DB hub]]** · **[[../../spring/java-spring/database/mybatis|↗ Java MyBatis 원본]]**

> 파일 이름은 `mybatis.md` 지만 Python 진영의 **raw SQL** 패턴. SQLAlchemy 의 `text()` 또는 Core SQL builder.

---

## 1. Python 진영의 MyBatis 대응 도구

| | MyBatis (Java) | Python 대응 |
| --- | --- | --- |
| SQL 직접 + 매핑 | XML mapper | `db.session.execute(text(...))` + dict |
| Dynamic SQL | `<if>`, `<foreach>` | Python `if` + parameter binding |
| Type-safe SQL builder | (없음 — XML) | **SQLAlchemy Core** (`select()`, `insert()`) |
| Generated SQL DSL | (없음) | (없음 — Java 의 JOOQ 동등) |

→ **Python 표준** = `text()` + Core 의 혼용. **SQL injection 방지가 핵심**.

---

## 2. text() — 가장 단순한 raw SQL

```python
from sqlalchemy import text
from app.extensions import db


def find_user_by_email(email: str) -> dict | None:
    sql = text("""
        SELECT id, email, name, status, created_at
        FROM users
        WHERE lower(email) = lower(:email)
    """)
    row = db.session.execute(sql, {"email": email}).mappings().first()
    return dict(row) if row else None


def search_users(email_pattern: str | None, limit: int, offset: int) -> list[dict]:
    """Dynamic SQL — Python conditional 으로 build."""
    where_clauses = ["status = 'ACTIVE'"]
    params = {"limit": limit, "offset": offset}

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
    rows = db.session.execute(sql, params).mappings().all()
    return [dict(r) for r in rows]
```

> **Python idiom 포인트**:
> - `text("SELECT ... WHERE x = :x")` — `:x` named parameter (PreparedStatement 동등).
> - `.execute(sql, {"x": value}).mappings()` — `Row` 대신 dict-like 반환.
> - **`{value}` (f-string) 사용 금지 — SQL injection**. parameter binding 만.

### 2.1 f-string 의 안전한 사용

WHERE 절 조립 같이 컬럼명 / 정적 SQL fragment 는 f-string OK:

```python
sort_col = "created_at" if sort_by == "LATEST" else "id"
# 화이트리스트 검증 후
assert sort_col in ("created_at", "id", "email"), "Invalid sort"

sql = text(f"SELECT * FROM users ORDER BY {sort_col} DESC LIMIT :limit")
```

> **사용자 입력은 절대 f-string X**. 화이트리스트 검증 후만.

---

## 3. SQLAlchemy Core — type-safe SQL builder

```python
from sqlalchemy import select, insert, update, delete, and_, or_

from app.domain.user import User


# 단순 SELECT
stmt = select(User).where(User.email == email)
user = db.session.scalars(stmt).first()

# 복잡 WHERE
stmt = (
    select(User)
    .where(
        and_(
            User.status == "ACTIVE",
            or_(
                User.email.ilike(f"%{pattern}%"),
                User.name.ilike(f"%{pattern}%"),
            ),
        ),
    )
    .order_by(User.created_at.desc())
    .limit(20)
)
users = db.session.scalars(stmt).all()

# JOIN
from sqlalchemy.orm import aliased
Memo = aliased(Memo)
stmt = (
    select(User, func.count(Memo.id).label("memo_count"))
    .join(Memo, Memo.owner_id == User.id, isouter=True)
    .group_by(User.id)
)
results = db.session.execute(stmt).all()
for user, count in results:
    print(user.email, count)

# 일괄 update
stmt = update(User).where(User.id.in_(ids)).values(status="INACTIVE")
result = db.session.execute(stmt)
db.session.commit()
print(result.rowcount)
```

> **Core 의 장점**: type-safe + Python 문법으로 dynamic. MyBatis 의 `<if>` `<choose>` 보다 더 명확.

---

## 4. Mapper 패턴 (MyBatis 의 `@Mapper` 대응)

Python 에는 어노테이션 mapper 없음. **class 또는 module-level 함수** 로 정리:

```python
# app/repositories/user_mapper.py
from sqlalchemy import text
from app.extensions import db


class UserMapper:
    """MyBatis @Mapper interface 동등 — SQL 모음."""

    def find_by_id(self, user_id: int) -> dict | None:
        sql = text("""
            SELECT id, email, name, password_hash AS passwordHash, status,
                   created_at AS createdAt, updated_at AS updatedAt
            FROM users WHERE id = :id
        """)
        row = db.session.execute(sql, {"id": user_id}).mappings().first()
        return dict(row) if row else None

    def insert(self, email: str, password_hash: str, name: str) -> int:
        sql = text("""
            INSERT INTO users (email, password_hash, name, status, created_at, updated_at)
            VALUES (:email, :password_hash, :name, 'ACTIVE', now(), now())
            RETURNING id
        """)
        result = db.session.execute(sql, {
            "email": email, "password_hash": password_hash, "name": name,
        })
        new_id = result.scalar_one()
        db.session.commit()
        return new_id

    def update(self, user_id: int, **fields) -> int:
        """동적 SQL — 명시한 필드만."""
        if not fields:
            return 0
        set_clauses = []
        params = {"id": user_id}
        for key, value in fields.items():
            assert key in {"email", "name", "status"}, f"Invalid field: {key}"
            set_clauses.append(f"{key} = :{key}")
            params[key] = value

        sql = text(f"""
            UPDATE users
            SET {", ".join(set_clauses)}, updated_at = now()
            WHERE id = :id
        """)
        result = db.session.execute(sql, params)
        db.session.commit()
        return result.rowcount
```

---

## 5. 결과 매핑 — `mappings()` vs `scalars()`

```python
# 1. mappings() — dict-like Row
rows = db.session.execute(sql, params).mappings().all()
# [{"id": 1, "email": "a@b.com", ...}, ...]

# 2. scalars() — 첫 컬럼만 (또는 ORM 객체)
ids = db.session.execute(text("SELECT id FROM users")).scalars().all()
# [1, 2, 3]

# 3. all() — tuple (Row)
rows = db.session.execute(sql).all()
# [(1, "a@b.com", ...), (2, ...)]
```

→ **dict / dataclass 변환은 `mappings()`** 권장. 객체 매핑은 `scalars(select(User))`.

---

## 6. 페이지네이션

### 6.1 직접 LIMIT / OFFSET

```python
def search(criteria: dict, limit: int = 20, offset: int = 0) -> list[dict]:
    sql = text("""
        SELECT * FROM users
        WHERE status = :status
        ORDER BY created_at DESC
        LIMIT :limit OFFSET :offset
    """)
    return [dict(r) for r in db.session.execute(sql, {
        **criteria, "limit": limit, "offset": offset,
    }).mappings().all()]
```

### 6.2 Cursor (keyset)

```python
def search_after(
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
    return [dict(r) for r in db.session.execute(sql, params).mappings().all()]
```

---

## 7. 1:N 결과 매핑

MyBatis 의 `<collection>` 동등 — Python 에서 직접:

```python
def find_product_with_options(product_id: int) -> dict | None:
    sql = text("""
        SELECT
            p.id AS p_id, p.name AS p_name,
            po.id AS po_id, po.name AS po_name, po.stock AS po_stock
        FROM products p
        LEFT JOIN product_options po ON po.product_id = p.id
        WHERE p.id = :id
    """)
    rows = db.session.execute(sql, {"id": product_id}).mappings().all()
    if not rows:
        return None

    # 첫 row 에서 product 정보
    first = rows[0]
    product = {"id": first["p_id"], "name": first["p_name"], "options": []}

    for r in rows:
        if r["po_id"]:
            product["options"].append({
                "id": r["po_id"], "name": r["po_name"], "stock": r["po_stock"],
            })
    return product
```

> **함정 — cartesian**: N options × M images 면 N×M row 폭발. **별도 select 권장**.

---

## 8. 트랜잭션

ORM 과 동일 — `db.session.commit()` / `db.session.rollback()`.

```python
def transfer_funds(from_id: int, to_id: int, amount: int) -> None:
    try:
        db.session.execute(
            text("UPDATE accounts SET balance = balance - :amt WHERE id = :id"),
            {"amt": amount, "id": from_id},
        )
        db.session.execute(
            text("UPDATE accounts SET balance = balance + :amt WHERE id = :id"),
            {"amt": amount, "id": to_id},
        )
        db.session.commit()
    except Exception:
        db.session.rollback()
        raise
```

---

## 9. typeHandler 대응 — converter

SQLAlchemy 는 native type 풍부. PG `jsonb` / `uuid` / `array` 자동:

```python
from sqlalchemy.dialects.postgresql import JSONB, UUID, ARRAY

class Event(db.Model):
    metadata_json: Mapped[dict] = mapped_column(JSONB)
    user_ids: Mapped[list[int]] = mapped_column(ARRAY(Integer))
```

raw `text()` 에서도 자동 변환:

```python
sql = text("SELECT metadata FROM events WHERE id = :id")
row = db.session.execute(sql, {"id": 1}).mappings().first()
print(row["metadata"])               # 이미 dict
```

---

## 10. 함정 모음

### 함정 1 — f-string 으로 user input 삽입
SQL injection 즉시. **무조건 `text(":x")` + parameter binding**.

### 함정 2 — `text(...)` 의 `:name` parameter 누락
binding 안 하면 SQLAlchemy 가 `:name` 을 SQL 일부로 보냄 → DB syntax error.

### 함정 3 — `.first()` vs `.one()` vs `.scalar()`
- `.first()` — 0 또는 1 (None 가능)
- `.one()` — 정확히 1 (없으면 NoResultFound)
- `.one_or_none()` — 0 또는 1 명시
- `.scalar()` — 단일 컬럼 단일 row (예: COUNT)

**의도 명확히**.

### 함정 4 — `mappings()` vs `all()` 혼동
`all()` = tuple, `mappings().all()` = dict-like. **반환 형식에 따라 명시**.

### 함정 5 — `<foreach>` 동등 (IN 절 대량)
Python 에선 `User.email.in_(emails)` 또는 `text("WHERE email IN :emails").bindparams(...)`:
```python
from sqlalchemy import bindparam
sql = text("SELECT * FROM users WHERE id IN :ids").bindparams(
    bindparam("ids", expanding=True),
)
db.session.execute(sql, {"ids": [1, 2, 3]})
```

### 함정 6 — `RETURNING` 이 PG / Oracle 만
MySQL 은 `result.lastrowid`. 또는 별도 SELECT.

### 함정 7 — Python 한정: f-string SQL injection 의 미묘한 케이스
```python
sql = text(f"SELECT * FROM users WHERE id = {user_id}")     # ❌
sql = text("SELECT * FROM users WHERE id = :id"); execute(sql, {"id": user_id})    # ✅
```
**user_id 가 int 라고 안심 X** — 의도 모호. 항상 parameter binding.

### 함정 8 — Python 한정: `text()` 의 escape
SQL 안 `'` 직접 쓰는 일 X (parameter binding 으로 처리). `text()` 안 f-string 으로 `'` 넣지 말 것.

---

## 11. JPA vs MyBatis (Java) — Flask 동등 비교

```python
# JPA 식 (SQLAlchemy ORM)
user = db.session.get(User, user_id)
user.name = "new"
db.session.commit()                  # dirty checking

# MyBatis 식 (raw SQL)
db.session.execute(
    text("UPDATE users SET name = :name WHERE id = :id"),
    {"name": "new", "id": user_id},
)
db.session.commit()
```

→ **단순 / 복잡 SQL** = ORM. **복잡 / 보고서 / tuning** = raw SQL.

---

## 12. 운영 체크리스트

- [ ] 모든 user input 이 parameter binding (`:x`)
- [ ] f-string SQL 은 화이트리스트 검증 후만 (정적 fragment)
- [ ] `mappings()` 형식 통일
- [ ] `IN :ids` 는 `bindparam(expanding=True)`
- [ ] PG `RETURNING` 활용 (insert 후 id)
- [ ] dev/staging — SQL log 활성

---

## 13. 관련

- [[../../spring/java-spring/database/mybatis|↗ Java MyBatis 원본]]
- [[jpa]] — SQLAlchemy ORM (JPA 대응)
- [[orm-coexist|jpa-mybatis-coexist]] — ORM + raw 함께
- [[database|↑ database hub]]
