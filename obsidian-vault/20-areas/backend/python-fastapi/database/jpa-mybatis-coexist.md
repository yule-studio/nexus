---
title: "SQLAlchemy ORM + Raw SQL async 공존 — FastAPI"
kind: knowledge
status: current
created_at: 2026-05-27T01:15:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, database, sqlalchemy, raw-sql, coexistence, async]
---

# SQLAlchemy ORM + Raw SQL async 공존 — FastAPI

**[[database|↑ DB hub]]** · **[[../../spring/java-spring/database/jpa-mybatis-coexist|↗ Java 원본]]** · **[[../../python-flask/database/jpa-mybatis-coexist|↗ Flask 변형 (sync)]]**

> Flask 변형의 패턴 동일 — `db.session` → `AsyncSession`, `db.session.flush()` → `await db.flush()`.

---

## 1-2-5. 왜 같이 쓰나 / 핵심 원칙 / 같은 테이블 분담 표

(Java 원본 [[../../spring/java-spring/database/jpa-mybatis-coexist#1|§1, §2, §5]] 와 동일)

---

## 3. 설정 — 별도 없음

```python
# app/db.py — async engine 하나
engine = create_async_engine(settings.DATABASE_URL)
AsyncSessionLocal = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)


async def get_db():
    async with AsyncSessionLocal() as session:
        yield session
```

ORM (`select(X)`) + raw SQL (`text(...)`) 둘 다 같은 `AsyncSession` 사용.

---

## 4. 역할 분담 패턴 (async)

```python
# ORM repository — CUD + 단순 조회
class UserRepository:
    async def save(self, db: AsyncSession, user: User) -> User:
        db.add(user)
        await db.commit()
        return user

    async def find_by_id(self, db: AsyncSession, user_id: int) -> User | None:
        return await db.get(User, user_id)


# raw SQL repository — 복잡 검색 / 보고서
class UserQueryRepository:
    async def search(
        self, db: AsyncSession, criteria: dict, page: int, size: int,
    ) -> dict:
        sql = text("""
            SELECT u.id, u.email, u.name,
                   COUNT(m.id) AS memo_count
            FROM users u
            LEFT JOIN memos m ON m.owner_id = u.id
            WHERE u.status = 'ACTIVE'
              AND (:email IS NULL OR u.email ILIKE concat('%', :email, '%'))
            GROUP BY u.id, u.email, u.name
            ORDER BY memo_count DESC
            LIMIT :limit OFFSET :offset
        """)
        result = await db.execute(sql, {
            **criteria, "limit": size, "offset": page * size,
        })
        return {"items": [dict(r) for r in result.mappings().all()]}

    async def monthly_stats(self, db: AsyncSession, year: int) -> list[dict]:
        sql = text("""
            SELECT
                date_trunc('month', created_at) AS month,
                COUNT(*) AS signup_count
            FROM users
            WHERE EXTRACT(year FROM created_at) = :year
            GROUP BY month
            ORDER BY month
        """)
        result = await db.execute(sql, {"year": year})
        return [dict(r) for r in result.mappings().all()]
```

---

## 5. dirty read 함정 (async)

```python
# ❌
async def transfer_and_summary(db: AsyncSession, user_id: int):
    user = await db.get(User, user_id)
    user.name = "changed"                    # ORM 변경

    # 같은 트랜잭션 raw SQL → 옛 name 보임
    sql = text("SELECT name FROM users WHERE id = :id")
    row = (await db.execute(sql, {"id": user_id})).mappings().first()
    print(row["name"])                       # 옛 값


# ✅ flush 명시
async def transfer_and_summary(db: AsyncSession, user_id: int):
    user = await db.get(User, user_id)
    user.name = "changed"
    await db.flush()                         # UPDATE 즉시 발사

    sql = text("SELECT name FROM users WHERE id = :id")
    row = (await db.execute(sql, {"id": user_id})).mappings().first()
    print(row["name"])                       # 새 값
```

---

## 6. 트랜잭션 — ORM + raw 같이

```python
async def place_order(db: AsyncSession, user_id: int, items: list[dict]) -> Order:
    try:
        # 1. ORM
        order = Order(user_id=user_id, total=sum(i["price"] for i in items))
        db.add(order)
        await db.flush()                     # order.id 받기

        # 2. raw SQL — audit
        await db.execute(
            text("INSERT INTO audit_logs (action, entity_id, created_at) "
                 "VALUES ('ORDER_PLACED', :id, now())"),
            {"id": order.id},
        )

        # 3. raw SQL — notification
        await db.execute(
            text("INSERT INTO notifications (user_id, type, data) "
                 "VALUES (:uid, 'ORDER_CREATED', :data::jsonb)"),
            {"uid": user_id, "data": json.dumps({"orderId": order.id})},
        )

        await db.commit()
        return order
    except Exception:
        await db.rollback()
        raise
```

---

## 7. 함정 모음

(Java 원본 [[../../spring/java-spring/database/jpa-mybatis-coexist#9 함정 모음|§9]] + Flask 변형 함정)

### FastAPI/async 한정

### 함정 1 — `await db.flush()` 안 호출 + 같은 session raw SQL
**flush() 명시**.

### 함정 2 — `await` 누락
`db.execute(sql)` 만 호출 → Coroutine 그대로 반환. 모든 호출 await.

### 함정 3 — BackgroundTasks 의 새 session
main request session 사용 X. **`async with AsyncSessionLocal() as bg_db:`** 새로.

### 함정 4 — 도메인 reconstruct factory 가 두 곳
ORM mapper 와 raw SQL dict 양쪽에서 변환 코드 중복. 한 factory.

---

## 8. 운영 체크리스트

(Flask 변형 §9 + async)

- [ ] 단일 async engine
- [ ] `await db.flush()` 명시 (ORM 변경 + raw read 시)
- [ ] 모든 호출 `await`
- [ ] BackgroundTasks 새 session
- [ ] ORM / raw 패키지 분리

---

## 9. 관련

- [[../../spring/java-spring/database/jpa-mybatis-coexist|↗ Java 원본]]
- [[../../python-flask/database/jpa-mybatis-coexist|↗ Flask 변형 (sync)]]
- [[jpa]] / [[mybatis]]
- [[database|↑ database hub]]
