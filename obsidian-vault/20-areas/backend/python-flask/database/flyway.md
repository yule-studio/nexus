---
title: "Alembic / Flask-Migrate — Flask DB 마이그레이션"
kind: knowledge
status: current
created_at: 2026-05-26T22:35:00+09:00
agent: engineering-agent/tech-lead
tags: [knowledge, python-flask, alembic, flask-migrate, migration, database]
home_hub: database
related:
  - "[[database]]"
  - "[[sqlalchemy]]"
  - "[[../python-flask]]"
  - "[[../../spring/java-spring/database/flyway]]"
---

# Alembic / Flask-Migrate — Flask DB 마이그레이션

**[[database|↑ DB hub]]** · **[[../../spring/java-spring/database/flyway|↗ Java Flyway]]**

> 노트 이름은 vault 전체 통일 위해 `flyway` 지만, Python/Flask 에선 **Alembic + Flask-Migrate** 가 동등 도구.

---

## 1. 왜 쓰는가

(Java 원본 [[../../spring/java-spring/database/flyway#1 왜 쓰는가|§1]] 와 동일 원칙: `db.create_all()` 운영 X. 마이그레이션 파일이 진실.)

Python/Flask 한정 추가:
- SQLAlchemy 의 `db.create_all()` = Java 의 `ddl-auto=create-drop` 과 동등 — **운영 X**.
- `Base.metadata.drop_all()` = 사고. 별도 명시적 admin script 만.

---

## 2. 의존성

```bash
pip install Flask-Migrate alembic
```

```python
# app/extensions.py
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate

db = SQLAlchemy()
migrate = Migrate()
```

```python
# app/__init__.py
def create_app():
    app = Flask(__name__)
    # ...
    db.init_app(app)
    migrate.init_app(app, db)
    # ...
```

---

## 3. 첫 사용

```bash
export FLASK_APP=wsgi.py

# 1. migrations 디렉터리 초기화 (한 번만)
flask db init

# 2. 모델 변경 후 — autogenerate
flask db migrate -m "create users and memos"

# 3. DB 에 적용
flask db upgrade

# 4. 롤백
flask db downgrade -1

# 5. 현재 버전 확인
flask db current

# 6. 히스토리
flask db history
```

생성된 디렉터리:
```
migrations/
├── alembic.ini
├── env.py                     # SQLAlchemy URL 연결
├── README
├── script.py.mako              # template
└── versions/
    ├── abc1234_create_users_and_memos.py
    └── def5678_add_phone.py
```

---

## 4. 파일 명명 규칙

```
versions/
├── 0001_init_schema.py
├── 0002_add_phone.py
└── 2026_05_20_001_add_payment.py     # 협업 시 날짜 prefix 권장
```

Alembic 의 revision id 는 자동 hex 12자리. 명시적 ID 면 `flask db revision --rev-id "0001"` 옵션.

---

## 5. 운영 패턴

### 5.1 PR 마다 마이그레이션 review

DDL 변경은 코드 review 대상. `alembic upgrade` 의 SQL 명확.

### 5.2 운영 deploy 순서

```
1. CI build → Docker image
2. CD deploy → 컨테이너 startup hook 에서 flask db upgrade
3. 새 코드 실행
```

```dockerfile
CMD ["sh", "-c", "flask db upgrade && gunicorn -w 4 -b 0.0.0.0:5000 wsgi:app"]
```

### 5.3 multi-instance 동시 startup

여러 인스턴스가 동시에 `flask db upgrade` = lock 경쟁. Alembic 의 `alembic_version` 테이블이 advisory lock 자동 — 한 인스턴스만 migrate, 나머지는 대기.

단 timeout 위험 — **CI job 1개에서만** migration 권장 + 앱 startup 은 skip.

### 5.4 autogenerate 의 한계

`flask db migrate` 가 자동 생성하지만:
- **컬럼 rename** 못 잡음 (drop + add 로 인식)
- **type 변경** 위험 (data 손실 가능)
- **constraint 변경** 일부 못 잡음

→ **항상 생성된 마이그레이션 file 검토 + 필요 시 수정**.

---

## 6. 함정 모음

### 함정 1 — `db.create_all()` 운영
DB 깨끗해 보이지만 마이그레이션 history 누락 → 다음 변경부터 alembic 충돌.

### 함정 2 — 같은 parent 의 마이그레이션 동시 생성
dev A, dev B 동시 작업 → 같은 down_revision 가진 V_a, V_b 두 파일. `flask db upgrade` 실패.
**해결**: `flask db merge -m "merge a and b" <rev_a> <rev_b>` 로 merge revision 생성.

### 함정 3 — env.py 의 `target_metadata` 누락
autogenerate 가 빈 마이그레이션 생성. 모든 모델이 import 되어야 metadata 인지.

### 함정 4 — `sqlalchemy.url` 가 env 가 아닌 alembic.ini 에 하드코딩
다른 환경 (staging / prod) 에서 잘못된 DB 에 마이그레이션. **env.py 에서 동적 주입**:
```python
# migrations/env.py
import os
config.set_main_option("sqlalchemy.url", os.environ["DATABASE_URL"])
```

### 함정 5 — 큰 테이블의 `ALTER TABLE`
PG / MySQL 의 `ALTER` 가 테이블 lock — 운영 시 service 중단. **PG 의 경우 `concurrently` index 또는 점진적 마이그레이션 (column add + nullable + backfill + not null) 분리**.

### 함정 6 — Alembic autogenerate 가 `server_default` 변경 못 잡음
명시적 `op.alter_column(...)` 추가.

### 함정 7 — Python 한정: `flask db upgrade` 가 SQLAlchemy import 무한 루프
모델 파일이 다른 모델 import 시 circular → autogenerate fail. **`if TYPE_CHECKING:`** 사용:
```python
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from .user import User
```

### 함정 8 — Python 한정: H2 같은 SQLite in-memory 와 PG dialect 차이
테스트는 SQLite, 운영은 PG → autogenerate 가 PG type (`JSONB`, `ARRAY`) 만 인식 못 함. **테스트도 docker PG** 권장.

### 함정 9 — Python 한정: Alembic env.py 가 sync 만 default
SQLAlchemy async (FastAPI) 면 `alembic init -t async` 로 init.

---

## 7. cross-link

- [[database]]
- [[sqlalchemy]] — 모델 + session 본편
- [[../../spring/java-spring/database/flyway|↗ Java Flyway]]
- [[../python-flask]]
- [[../getting-started#4 첫 마이그레이션|getting-started §4.4]] — 첫 마이그레이션 step
