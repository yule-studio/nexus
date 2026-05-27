---
title: "02. 프로젝트 구조 + app factory + Blueprint + .env"
kind: guide
status: current
created_at: 2026-05-27T12:20:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-flask, guidelines, structure, blueprint, config]
---

# 02. 프로젝트 구조 + app factory + Blueprint + .env

**[[guidelines|↑ Hub]]** ← [[01-installation-and-hello-world]] · → 다음 [[03-domain-and-sqlalchemy]]

> 메모 API 본격 프로젝트 패키지 구조 + app factory + Blueprint + Config + .env.

---

## 1. 권장 디렉토리

```
flask-memo-api/
├── .venv/                          # gitignore
├── .env                            # gitignore (secret)
├── .gitignore
├── requirements.txt
├── pyproject.toml                  # (선택) modern config
├── README.md
├── app/
│   ├── __init__.py                 # create_app() factory
│   ├── config.py                   # Config 클래스
│   ├── extensions.py               # SQLAlchemy / JWT / Migrate 객체
│   ├── domain/                     # POPO + 도메인 로직
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── memo.py
│   ├── repositories/               # data access
│   │   └── ...
│   ├── services/                   # business logic
│   │   ├── auth_service.py
│   │   └── memo_service.py
│   ├── schemas/                    # marshmallow request/response
│   │   ├── auth.py
│   │   └── memo.py
│   ├── api/                        # Flask Blueprint
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   └── memos.py
│   ├── common/                     # envelope / exception / responses
│   │   ├── responses.py
│   │   ├── exceptions.py
│   │   └── error_handlers.py
│   └── workers/                    # Celery (선택)
├── migrations/                     # Alembic (Flask-Migrate)
├── tests/
│   ├── conftest.py
│   └── test_*.py
└── wsgi.py                         # entry point — gunicorn 이 import
```

---

## 2. `.gitignore`

```gitignore
.venv/
__pycache__/
*.pyc
.env
.env.*
!.env.example
.pytest_cache/
.mypy_cache/
.coverage
htmlcov/
dist/
*.egg-info/
.DS_Store
instance/
```

---

## 3. `requirements.txt`

```txt
flask==3.0.2
Flask-SQLAlchemy==3.1.1
Flask-Migrate==4.0.5
Flask-JWT-Extended==4.6.0
Flask-CORS==4.0.1
Flask-Caching==2.1.0
Flask-Limiter==3.5.1
marshmallow==3.21.1
argon2-cffi==23.1.0
psycopg[binary]==3.1.18
python-dotenv==1.0.1
gunicorn==21.2.0
```

`pip install -r requirements.txt`.

---

## 4. `.env` + Config

```bash
# .env (gitignore)
FLASK_ENV=development
SECRET_KEY=dev-secret-change-in-prod
JWT_SECRET_KEY=dev-jwt-secret-change-in-prod
DATABASE_URL=postgresql+psycopg://memo:memo@localhost:5432/memo
CORS_ORIGINS=http://localhost:3000
```

```bash
# .env.example (commit) — 동료가 어떤 변수 필요한지 알도록
FLASK_ENV=
SECRET_KEY=
JWT_SECRET_KEY=
DATABASE_URL=
CORS_ORIGINS=
```

```python
# app/config.py
import os
from datetime import timedelta
from dotenv import load_dotenv

load_dotenv()


class Config:
    SECRET_KEY = os.environ["SECRET_KEY"]
    SQLALCHEMY_DATABASE_URI = os.environ["DATABASE_URL"]
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    SQLALCHEMY_ENGINE_OPTIONS = {
        "pool_size": 20,
        "max_overflow": 10,
        "pool_pre_ping": True,
        "pool_recycle": 1800,
    }

    JWT_SECRET_KEY = os.environ["JWT_SECRET_KEY"]
    JWT_ACCESS_TOKEN_EXPIRES = timedelta(minutes=15)
    JWT_REFRESH_TOKEN_EXPIRES = timedelta(days=14)

    CORS_ORIGINS = os.environ.get("CORS_ORIGINS", "http://localhost:3000").split(",")


class TestConfig(Config):
    TESTING = True
    SQLALCHEMY_DATABASE_URI = "sqlite:///:memory:"      # 빠른 테스트
    SECRET_KEY = "test-secret"
    JWT_SECRET_KEY = "test-jwt-secret"


class ProductionConfig(Config):
    # 운영 추가 설정
    DEBUG = False
```

> **`load_dotenv()`**: `.env` 파일을 `os.environ` 으로 로드. 운영에선 시스템 env 직접 사용.
> **함정**: `os.environ["X"]` 가 미설정 시 import-time KeyError → docker 무한 restart. `.env` 안 보내면 컨테이너 startup fail.

---

## 5. Extensions — 객체 한 곳에

```python
# app/extensions.py
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_jwt_extended import JWTManager
from flask_cors import CORS
from flask_caching import Cache
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

db = SQLAlchemy()
migrate = Migrate()
jwt = JWTManager()
cors = CORS()
cache = Cache()
limiter = Limiter(key_func=get_remote_address)
```

> **왜 분리**: `create_app` 안에 다 두면 circular import 위험. 객체는 미리, `init_app(app)` 으로 나중 연결.

---

## 6. App factory

```python
# app/__init__.py
from flask import Flask

from .config import Config
from .extensions import db, migrate, jwt, cors, cache, limiter


def create_app(config_class=Config) -> Flask:
    app = Flask(__name__)
    app.config.from_object(config_class)

    # 확장 init
    db.init_app(app)
    migrate.init_app(app, db)
    jwt.init_app(app)
    cors.init_app(
        app,
        resources={r"/api/*": {"origins": app.config["CORS_ORIGINS"]}},
        supports_credentials=False,
    )
    cache.init_app(app, config={"CACHE_TYPE": "SimpleCache"})
    limiter.init_app(app)

    # Blueprint 등록
    from .api.auth import auth_bp
    from .api.memos import memos_bp
    app.register_blueprint(auth_bp)
    app.register_blueprint(memos_bp)

    # 에러 핸들러
    from .common.error_handlers import register_error_handlers
    register_error_handlers(app)

    # JWT loader (envelope 통일) — 05-security 참고
    from .common.jwt_loaders import register_jwt_loaders
    register_jwt_loaders(jwt)

    # 모델 import — Flask-Migrate 가 인지
    from .domain import user, memo  # noqa: F401

    return app
```

> **`create_app` 패턴 장점**:
> - testing / dev / prod 마다 다른 Config 주입
> - circular import 회피
> - 여러 app 인스턴스 동시 생성 (테스트 격리)

---

## 7. Blueprint — Flask 의 module 단위

```python
# app/api/auth.py
from flask import Blueprint

auth_bp = Blueprint("auth", __name__, url_prefix="/api/v1/auth")


@auth_bp.post("/signup")
def signup():
    # ...
    pass


@auth_bp.post("/login")
def login():
    # ...
    pass
```

```python
# app/api/memos.py
from flask import Blueprint

memos_bp = Blueprint("memos", __name__, url_prefix="/api/v1/memos")


@memos_bp.get("")
def list_memos():
    # ...
    pass


@memos_bp.post("")
def create_memo():
    # ...
    pass
```

→ `app.register_blueprint(auth_bp)` 로 등록. URL 자동 prefix.

### 7.1 Blueprint 의 url_prefix

```python
# Blueprint 단에 prefix
auth_bp = Blueprint("auth", __name__, url_prefix="/api/v1/auth")

# 또는 register 시 prefix
app.register_blueprint(auth_bp, url_prefix="/api/v2/auth")     # 동적
```

### 7.2 Blueprint 단 error handler

```python
@auth_bp.errorhandler(BusinessException)
def handle_business(ex):
    return {"error": ex.message}, ex.status
```

→ 해당 Blueprint 의 endpoint 에서 발생한 예외만. 전역은 `@app.errorhandler`.

---

## 8. entry point — `wsgi.py`

```python
# wsgi.py (project root)
from app import create_app

app = create_app()

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
```

```bash
# dev
python wsgi.py
# 또는
flask --app wsgi run --debug

# 운영
gunicorn -w 4 -b 0.0.0.0:5000 wsgi:app
```

---

## 9. 환경별 분리 — `FLASK_CONFIG` 또는 `FLASK_ENV`

```python
# app/__init__.py 변형
def create_app(config_class=None) -> Flask:
    app = Flask(__name__)

    if config_class is None:
        config_name = os.environ.get("FLASK_CONFIG", "Config")
        config_class = {
            "Config": Config,
            "TestConfig": TestConfig,
            "ProductionConfig": ProductionConfig,
        }.get(config_name, Config)

    app.config.from_object(config_class)
    # ...
    return app
```

```bash
# 운영
export FLASK_CONFIG=ProductionConfig
gunicorn -w 4 -b 0.0.0.0:5000 wsgi:app
```

---

## 10. logging 기본

```python
# app/__init__.py 끝부분
import logging

def create_app(...):
    app = Flask(__name__)
    # ...
    if not app.debug:
        logging.basicConfig(level=logging.INFO)
        app.logger.setLevel(logging.INFO)
    return app
```

자세한 logging 은 [[09-logging-mdc]].

---

## 11. 자주 막히는 함정

### 함정 1 — circular import
`extensions.py` 와 `domain/user.py` 가 서로 import → ImportError. **extensions 는 객체만**, model 은 domain 패키지에. 한 방향 import.

### 함정 2 — `from .extensions import db` 의 위치
`app/__init__.py` 의 함수 안 (local import) 이 안전. module-level 면 import 시점 문제.

### 함정 3 — Blueprint 의 `__name__` 가 module path 와 다름
`Blueprint("auth", __name__, ...)` 에서 `__name__` 가 `app.api.auth` 여야. 잘못된 이름 = static path 등 동작 X.

### 함정 4 — `os.environ["X"]` KeyError 로 import-time crash
docker 무한 restart → kubectl logs 만으론 원인 어려움. **`os.environ.get("X")` + None 체크** 또는 startup health check.

### 함정 5 — `.env` commit
secret 노출. **반드시 .gitignore**.

### 함정 6 — `SQLALCHEMY_TRACK_MODIFICATIONS=True` (default)
deprecation 경고 + 메모리 부담. **False 명시**.

### 함정 7 — Flask-SQLAlchemy 2.x vs 3.x 호환
3.x: `db.session.scalar(...)`, 2.x: `User.query.filter(...)` (deprecated). **3.x 권장**.

### 함정 8 — `app.app_context()` 안에서만 동작하는 코드를 밖에서
CLI / 테스트 / Celery 안 `db.session` 호출 시 `RuntimeError`. **`with app.app_context():`** 명시.

### 함정 9 — Blueprint URL `/path` vs `/path/`
Flask default 가 `strict_slashes=True` — `/x` 와 `/x/` 다름. 보통 `app.url_map.strict_slashes = False` 또는 일관된 규칙.

### 함정 10 — `requirements.txt` 의 `==` 없음
`pip install flask` 만으로 의존성 lock 안 됨. **`pip freeze`** 또는 `pip-compile`.

---

## 12. 다음

✅ 프로젝트 구조 + 설정 끝.

→ [[03-domain-and-sqlalchemy]] — SQLAlchemy 2.x 모델 + Flask-Migrate
