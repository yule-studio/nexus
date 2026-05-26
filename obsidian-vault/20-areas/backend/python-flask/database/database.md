---
title: "Python Flask — Data Access / ORM Hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T22:30:00+09:00
tags:
  - backend
  - python-flask
  - database
  - orm
  - hub
---

# Python Flask — Data Access / ORM Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 폴더 hub placeholder |
| v.2.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring/database hub 미러링 + Python idiom |

**[[../python-flask|↑ Flask]]** · **[[../../spring/java-spring/database/database|↗ Java 원본]]**

> 레시피 (`api-design/`) 는 Repository 단까지만. 구체 ORM 구현은 이 폴더가 본거지.

---

## 1. Python Flask 의 Data Access 4가지

| 도구 | 한 줄 | 강점 | 약점 |
| --- | --- | --- | --- |
| **SQLAlchemy ORM** | Python 의 표준 ORM (declarative + session) | 풍부 / 영속성 컨텍스트 / 마이그레이션 (Alembic) | 학습 곡선 / 마법 많음 |
| **SQLAlchemy Core** | SQL builder (table / select 등) | 타입 안전 + 직접 SQL 가까움 | ORM 의 dirty checking 없음 |
| **MyBatis 대응 — `db.session.execute(text(...))`** | raw SQL + dict 매핑 | SQL 가독성 / 튜닝 | 매핑 boilerplate |
| **Django ORM** | (Flask 가 아님 — Django 만) | — | — |

→ **Flask 표준** = **SQLAlchemy ORM + Alembic + 일부 raw SQL (복잡 보고서)**.

---

## 2. SQLAlchemy 2.x — sync vs async

| | sync | async |
| --- | --- | --- |
| Flask | ✅ 기본 (Flask-SQLAlchemy) | 약함 (Quart 권장) |
| FastAPI | OK | ✅ 기본 |
| 권장 | **Flask = sync** | FastAPI = async |

Flask 는 WSGI (sync) — async DB 쓰려면 Quart 또는 FastAPI. 본 가이드는 **sync SQLAlchemy 2.x**.

상세:
- [[sqlalchemy]] — SQLAlchemy 본편 (declarative / session / N+1 회피 — 예정)
- [[alembic]] — 마이그레이션 (Flask-Migrate wrapper, [[flyway]] 와 동등 — 본 노트 §5 참고)
- [[raw-sql]] — `db.session.execute(text(...))` 패턴 (예정)

---

## 3. 표준 의존성

```python
# requirements.txt
Flask-SQLAlchemy==3.1.1
Flask-Migrate==4.0.5         # Alembic wrapper
SQLAlchemy==2.0.29
alembic==1.13.1
psycopg[binary]==3.1.18      # PostgreSQL 드라이버
```

```yaml
# application config (Config 클래스)
SQLALCHEMY_DATABASE_URI: "postgresql+psycopg://memo:memo@localhost:5432/memo"
SQLALCHEMY_TRACK_MODIFICATIONS: False
SQLALCHEMY_ENGINE_OPTIONS:
  pool_size: 20
  max_overflow: 10
  pool_pre_ping: True              # connection 끊김 자동 감지
  pool_recycle: 1800               # 30분
  pool_timeout: 5                  # 풀 대기 timeout
```

---

## 4. 표준 구조 (Flask)

```
app/
├── extensions.py            # db = SQLAlchemy()
├── domain/
│   ├── user.py              # class User(db.Model)
│   └── memo.py
├── repositories/            # (선택) data 접근 격리
│   └── user_repo.py
└── services/                # business logic
```

(자세한 구조는 [[../getting-started#3 본격 프로젝트|getting-started §3]] 참고)

---

## 5. 마이그레이션 — Alembic / Flask-Migrate

본거지: [[flyway]] (이름은 Flyway 지만 Alembic 통합 가이드).

```bash
flask db init                    # 한 번만
flask db migrate -m "create users"
flask db upgrade                 # 적용

# rollback
flask db downgrade -1
```

> **운영 절대 규칙**: `db.create_all()` 운영에 X. 항상 `flask db upgrade`.

---

## 6. 이 폴더의 노트

```
database/
├── database.md              ← 본 hub
├── sqlalchemy.md            (예정) — SQLAlchemy 본편
├── alembic.md / flyway.md  ← 마이그레이션
├── raw-sql.md               (예정) — text() 패턴
└── connection-pool.md       (예정) — PgBouncer / Hikari 동등
```

진행:

| 노트 | 상태 |
| --- | --- |
| [[sqlalchemy]] | ✅ |
| [[flyway]] | ✅ (Alembic 통합 — 이름은 Java 와 통일) |
| [[raw-sql]] | ✅ |
| [[orm-coexist]] | ✅ |
| sqlalchemy-async | 🟡 (FastAPI 쪽) |

---

## 7. 레시피와의 연결

[[../api-design/api-design|api-design]] 의 각 레시피는 SQLAlchemy ORM 또는 raw SQL 로 구현 — 본 폴더 참조.

| 레시피 | 권장 구현 |
| --- | --- |
| 인증 / 회원가입 | SQLAlchemy ORM (CUD + 단순 조회) |
| [[../api-design/feed-timeline]] | raw SQL (cursor + 복잡 WHERE) |
| [[../api-design/review]] | ORM + raw SQL atomic UPDATE |
| [[../api-design/file-upload-s3]] | ORM (메타 저장만) |

---

## 8. Python / Flask 한정 운영·디버깅 체크리스트

- [ ] `pool_pre_ping=True` (connection 끊김 감지)
- [ ] `pool_recycle` 가 DB `wait_timeout` 보다 짧게
- [ ] gunicorn `workers × pool_size ≤ PG max_connections`
- [ ] `SQLALCHEMY_TRACK_MODIFICATIONS = False` (Flask-SQLAlchemy 의 deprecated 옵션)
- [ ] `flask db upgrade` 가 컨테이너 startup
- [ ] Alembic autogenerate 한 마이그레이션은 항상 PR review (자동 ≠ 정답)
- [ ] N+1 회피 — `selectinload` / `joinedload` / `.options(...)`
- [ ] dev/staging — SQL log (`SQLALCHEMY_ECHO = True` 또는 별도 logger)

---

## 9. 관련

- [[../../spring/java-spring/database/database|↗ Java 원본]]
- [[../api-design/api-design|↗ API 레시피 hub]]
- [[../../../../database/database|↗ 20-areas/database (DB 자체)]]
- [[../python-flask|↑ Flask]]
- [[../getting-started]] — Flask 입문 가이드
