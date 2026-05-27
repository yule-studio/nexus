---
title: "SQLAlchemy ORM + Raw SQL 공존 — Flask"
kind: knowledge
status: current
created_at: 2026-05-26T23:00:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-flask, database, sqlalchemy, raw-sql, coexistence]
---

# SQLAlchemy ORM + Raw SQL 공존 — Flask

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring/database/jpa-mybatis-coexist.md 미러링 |

**[[database|↑ DB hub]]** · **[[../../spring/java-spring/database/jpa-mybatis-coexist|↗ Java 원본]]**

> 파일 이름은 vault 통일을 위해 `jpa-mybatis-coexist.md`. 실제 내용은 **SQLAlchemy ORM + text() / Core (raw SQL) 함께**.
> Flask 진영의 표준 패턴 = ORM (CUD) + raw SQL (복잡 read).

---

## 1. 왜 같이 쓰나

(Java 원본 [[../../spring/java-spring/database/jpa-mybatis-coexist#1 왜 같이 쓰나|§1]] 의 정책 동일)

Python 한정 — SQLAlchemy 의 좋은 점:
- **같은 session / 같은 transaction** — Java 처럼 별도 SqlSession 설정 필요 X
- **Connection pool 자동 공유**
- **`db.session.execute(text(...))`** 가 ORM 과 같은 session 사용 → flush 의식

---

## 2. 핵심 원칙

1. **단일 DB engine** — Flask-SQLAlchemy `db = SQLAlchemy()` 한 개
2. **단일 session** — `db.session` 이 ORM + raw SQL 모두 사용
3. **`flush()` 의식** — ORM 변경 후 raw SQL 조회 시 dirty data 안 보임
4. **역할 분리** — 같은 테이블의 CUD 를 ORM + raw 양쪽에서 쓰지 말 것

---

## 3. 설정 — 별도 없음

```python
# app/extensions.py
from flask_sqlalchemy import SQLAlchemy
db = SQLAlchemy()           # ORM + raw 둘 다 같은 engine
```

Java 와 다르게 별도 SqlSessionFactory bean 등록 필요 없음.

---

## 4. 역할 분담 패턴

```python
# app/repositories/user_repo.py — ORM (CUD + 단순 조회)
class UserRepository:
    def save(self, user: User) -> User:
        db.session.add(user)
        db.session.commit()
        return user

    def find_by_id(self, user_id: int) -> User | None:
        return db.session.get(User, user_id)

    def find_by_email(self, email: str) -> User | None:
        stmt = select(User).where(User.email == email)
        return db.session.scalars(stmt).first()


# app/repositories/user_query_repo.py — raw SQL (복잡 검색 / 보고서)
class UserQueryRepository:
    def search(self, criteria: dict, page: int, size: int) -> dict:
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
        rows = db.session.execute(sql, {
            **criteria,
            "limit": size,
            "offset": page * size,
        }).mappings().all()
        return {"items": [dict(r) for r in rows]}

    def monthly_stats(self, year: int) -> list[dict]:
        sql = text("""
            SELECT
                date_trunc('month', created_at) AS month,
                COUNT(*) AS signup_count
            FROM users
            WHERE EXTRACT(year FROM created_at) = :year
            GROUP BY month
            ORDER BY month
        """)
        return [dict(r) for r in db.session.execute(sql, {"year": year}).mappings().all()]
```

→ Service 가 두 repository 모두 주입:

```python
class UserService:
    def __init__(self, repo: UserRepository, query: UserQueryRepository) -> None:
        self.repo = repo
        self.query = query

    def signup(self, email: str, ...) -> User:
        # ORM 사용
        user = User(email=email, ...)
        return self.repo.save(user)

    def admin_search(self, criteria: dict, page: int, size: int) -> dict:
        # raw SQL 사용
        return self.query.search(criteria, page, size)
```

---

## 5. 같은 트랜잭션 안 dirty read 함정

```python
# ❌ 잘못된 패턴
def transfer_and_summary(user_id: int):
    user = db.session.get(User, user_id)
    user.name = "changed"                    # ORM 변경 — 아직 flush X

    # 같은 트랜잭션에서 raw SQL 로 조회
    sql = text("SELECT name FROM users WHERE id = :id")
    row = db.session.execute(sql, {"id": user_id}).mappings().first()
    print(row["name"])                       # ⚠️ 옛 name (dirty 안 보임)


# ✅ 해결 — flush 명시
def transfer_and_summary(user_id: int):
    user = db.session.get(User, user_id)
    user.name = "changed"
    db.session.flush()                        # UPDATE 즉시 발사 (commit 아님)

    sql = text("SELECT name FROM users WHERE id = :id")
    row = db.session.execute(sql, {"id": user_id}).mappings().first()
    print(row["name"])                       # 새 name
```

> **autoflush=True (Flask-SQLAlchemy default)** 가 일부 케이스는 자동 flush. 그러나 안전하게 **명시적 `db.session.flush()`** 권장.

---

## 6. 같은 테이블 — 무엇을 ORM, 무엇을 raw

| 작업 | ORM | raw SQL |
| --- | --- | --- |
| 사용자 등록 (`POST /users`) | ✅ Aggregate CUD | ❌ |
| 사용자 조회 by id | ✅ `db.session.get(User, id)` | ❌ |
| 사용자 검색 (이메일 + paging) | ❌ | ✅ |
| 월별 가입자 통계 | ❌ | ✅ |
| 사용자 soft delete | ✅ entity 메서드 | ❌ |
| GDPR export (전체 JOIN) | ❌ | ✅ |

---

## 7. 트랜잭션 — ORM + raw 같이

```python
def place_order(user_id: int, items: list[dict]) -> Order:
    try:
        # 1. ORM 으로 Aggregate 저장
        order = Order(user_id=user_id, total=sum(i["price"] for i in items))
        db.session.add(order)
        db.session.flush()                   # order.id 받기

        # 2. raw SQL 로 audit log INSERT
        db.session.execute(
            text("INSERT INTO audit_logs (action, entity_id, created_at) "
                 "VALUES ('ORDER_PLACED', :id, now())"),
            {"id": order.id},
        )

        # 3. 알림 (raw SQL)
        db.session.execute(
            text("INSERT INTO notifications (user_id, type, data) "
                 "VALUES (:uid, 'ORDER_CREATED', :data::jsonb)"),
            {"uid": user_id, "data": json.dumps({"orderId": order.id})},
        )

        db.session.commit()
        return order
    except Exception:
        db.session.rollback()
        raise
```

→ 셋 다 한 트랜잭션. 하나라도 실패 = 모두 rollback.

---

## 8. 함정 모음

(Java 원본 [[../../spring/java-spring/database/jpa-mybatis-coexist#9 함정 모음|§9]] 의 함정 동일)

### Python/Flask 한정

### 함정 1 — `db.session.flush()` 안 호출 + raw SQL 조회
ORM 변경이 raw SQL 에 안 보임. **flush() 명시**.

### 함정 2 — `db.session.rollback()` 안 호출
한 트랜잭션의 ORM + raw 중 하나만 실패하고 rollback 안 하면 다음 query 모두 깨짐. **`try/except` 모두 rollback**.

### 함정 3 — raw SQL 의 SQL injection
ORM 만 쓸 때는 자연스럽지만 raw SQL 섞으면 user input 그대로 f-string 위험. **parameter binding 강제**.

### 함정 4 — Celery worker 에서 같은 session 공유 시도
worker thread 가 main thread session 못 쓰음. **`with app.app_context():` 안에서 새 session**.

### 함정 5 — 도메인 reconstruct 가 두 곳
ORM repository 의 `_to_domain` + raw repository 의 dict → domain object 변환이 별도 코드. **factory 한 곳**:
```python
@classmethod
def from_dict(cls, d: dict) -> "User":
    return cls(id=d["id"], email=d["email"], ...)
```

---

## 9. 운영 / 디버깅 체크리스트

- [ ] 단일 `db = SQLAlchemy()` engine (별도 설정 X)
- [ ] 같은 트랜잭션 안 ORM 변경 + raw SQL 조회 시 `db.session.flush()` 명시
- [ ] 도메인 reconstruct factory 한 곳
- [ ] ORM / raw 코드를 패키지 분리 (`repositories/orm/` vs `repositories/raw/`)
- [ ] 같은 테이블의 CUD 를 두 도구로 분산 금지
- [ ] raw SQL 의 모든 user input parameter binding

---

## 10. 관련

- [[../../spring/java-spring/database/jpa-mybatis-coexist|↗ Java 원본]]
- [[jpa]] / [[mybatis]]
- [[database|↑ database hub]]
