---
title: "Flask Getting Started — 0 부터 첫 프로젝트 배포까지"
kind: guide
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T19:00:00+09:00
tags:
  - backend
  - python-flask
  - getting-started
  - guide
  - tutorial
---

# Flask Getting Started — 0 부터 첫 프로젝트 배포까지

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | 설치 → 첫 프로젝트 → CRUD API → 배포 단일 가이드 |

**[[python-flask|↑ Flask]]**

> **이 가이드를 보고 처음부터 Flask 프로젝트를 만들고 배포까지 할 수 있도록 설계.**
> 사전 지식 가정 — Python 기본 문법은 안다 (`if/for/def/class`). HTTP / JSON / DB / Linux 명령어 약간.
> 끝나면 [[api-design/api-design|api-design]] 의 실전 레시피 또는 [[database/database|database]] 의 ORM 깊이로.

---

## 0. Flask 가 무엇인가

**Flask** = Python 의 **경량 (micro) 웹 프레임워크**. 한 함수가 한 HTTP endpoint.

```python
from flask import Flask

app = Flask(__name__)

@app.get("/hello")
def hello():
    return {"message": "Hello, World"}

# python app.py 로 실행하면 http://localhost:5000/hello 에서 JSON 응답.
if __name__ == "__main__":
    app.run(debug=True)
```

### Flask vs Django vs FastAPI — 한 줄

| | Flask | Django | FastAPI |
| --- | --- | --- | --- |
| 컨셉 | micro (필요한 것만 추가) | full-stack (admin / ORM / template 포함) | modern (async + type hints + auto OpenAPI) |
| 학습 곡선 | **가장 쉬움** | 가파름 (관습 많음) | 중간 (type hint 기본) |
| 적합 | 작은 API / prototype / 학습 | 큰 SaaS / admin 필요 / SI | 신규 마이크로서비스 / async 필수 |
| Spring 비유 | Spring 가장 raw 한 form | Spring Boot full-stack starter | Spring Boot + Reactor |

→ 본 가이드는 **Flask** 부터. 단순 + 명확.

### 0.1 무엇을 만들 건가

가상의 **메모 API 서비스** — 끝까지 따라하면:

```
GET    /api/v1/memos               # 목록
POST   /api/v1/memos               # 생성
GET    /api/v1/memos/{id}          # 단건
PATCH  /api/v1/memos/{id}          # 수정
DELETE /api/v1/memos/{id}          # 삭제

POST   /api/v1/auth/signup         # 회원가입
POST   /api/v1/auth/login          # JWT 발급
```

User 가 본인 메모만 CRUD. PostgreSQL + JWT 인증 + 배포.

---

## 1. 사전 준비 — Python / 가상환경 / Editor

### 1.1 Python 3.12 설치

**macOS**:
```bash
brew install python@3.12
python3.12 --version    # Python 3.12.x
```

**Ubuntu/Debian**:
```bash
sudo apt update
sudo apt install python3.12 python3.12-venv python3-pip
python3.12 --version
```

**Windows**: [python.org](https://www.python.org/downloads/) 에서 installer. 설치 시 `Add Python to PATH` 체크.

### 1.2 가상환경 (venv) — 프로젝트별 패키지 격리

Java/Kotlin 의 Gradle 의존성, JS 의 `node_modules` 와 동등. **반드시 프로젝트별로 분리**.

```bash
mkdir flask-memo-api && cd flask-memo-api
python3.12 -m venv .venv

# 활성화
source .venv/bin/activate           # macOS / Linux
# .venv\Scripts\activate            # Windows PowerShell

# 활성화 확인 — prompt 에 (.venv) 표시
which python                         # .venv/bin/python
python --version                     # Python 3.12.x
```

> **함정**: venv 활성화 안 하면 system Python 에 패키지 설치 → 다른 프로젝트 / OS 자체 깨질 수 있음. 매 세션마다 `source .venv/bin/activate`.

### 1.3 Editor

**VS Code** + Python 확장 (Microsoft) 권장. **PyCharm** 도 OK.

VS Code 의 `.venv` 자동 감지 — 화면 우하단 Python 버전 클릭 → `.venv/bin/python` 선택.

---

## 2. 첫 Flask app — Hello World

### 2.1 Flask 설치

```bash
pip install --upgrade pip
pip install flask
pip freeze > requirements.txt        # 의존성 lock
```

> `pip freeze` 결과:
> ```
> blinker==1.7.0
> click==8.1.7
> Flask==3.0.2
> itsdangerous==2.1.2
> Jinja2==3.1.3
> MarkupSafe==2.1.5
> Werkzeug==3.0.1
> ```

### 2.2 hello.py

```python
# hello.py
from flask import Flask

app = Flask(__name__)


@app.get("/")
def index():
    return {"message": "Hello, Flask!"}


@app.get("/hello/<name>")
def hello_name(name):
    return {"message": f"Hello, {name}!"}


if __name__ == "__main__":
    app.run(host="127.0.0.1", port=5000, debug=True)
```

### 2.3 실행

```bash
python hello.py
# * Serving Flask app 'hello'
# * Debug mode: on
# * Running on http://127.0.0.1:5000

# 다른 터미널
curl http://127.0.0.1:5000/
# {"message":"Hello, Flask!"}

curl http://127.0.0.1:5000/hello/yuchan
# {"message":"Hello, yuchan!"}
```

### 2.4 핵심 개념 — Route / View function / Decorator

- **`@app.get("/path")`** — 데코레이터. 뒤 함수가 GET `/path` 요청 처리.
- **View function** — `index()`, `hello_name(name)` 같은 endpoint 핸들러.
- **반환** `dict` → Flask 가 자동으로 JSON + `Content-Type: application/json`.
- **URL parameter** — `<name>` → 함수 인자.

`@app.get` / `@app.post` / `@app.put` / `@app.patch` / `@app.delete` 가 메서드별 단축형. 옛날엔 `@app.route("/path", methods=["GET"])` 였음.

> **Spring 비유**: `@app.get("/x")` = `@GetMapping("/x")`. View function = Controller method. dict 반환 = `ResponseEntity<Map>`.

---

## 3. 본격 프로젝트 — 구조 + Blueprint + App factory

`hello.py` 한 파일로 커지면 어지러워짐. **확장 가능한 구조** 로 재시작.

### 3.1 디렉터리

```
flask-memo-api/
├── .venv/                       # gitignore
├── .env                         # gitignore (secret 들)
├── .gitignore
├── requirements.txt
├── pyproject.toml               # (선택) modern Python config
├── README.md
├── app/
│   ├── __init__.py              # create_app() factory
│   ├── config.py                # Config 클래스
│   ├── extensions.py            # SQLAlchemy, JWTManager 등 객체 init
│   ├── domain/                  # 도메인 model (POPO)
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── memo.py
│   ├── api/                     # Flask Blueprint (HTTP 경계)
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   └── memos.py
│   ├── services/                # business logic
│   │   ├── __init__.py
│   │   ├── auth_service.py
│   │   └── memo_service.py
│   ├── schemas/                 # marshmallow / pydantic (입력 검증)
│   │   ├── __init__.py
│   │   ├── auth_schema.py
│   │   └── memo_schema.py
│   └── errors.py                # custom Exception + errorhandler
├── migrations/                  # Alembic (다음 §6)
└── tests/
    ├── conftest.py
    ├── test_auth.py
    └── test_memos.py
```

> **왜 이렇게**: Spring 의 `controller/service/repository/dto` 구조 동등. **HTTP 경계 (`api/`) ↔ business (`services/`) ↔ data (`domain/`) 분리**.

### 3.2 `.gitignore`

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
```

### 3.3 `requirements.txt`

```txt
flask==3.0.2
Flask-SQLAlchemy==3.1.1
Flask-Migrate==4.0.5
Flask-JWT-Extended==4.6.0
Flask-CORS==4.0.1
marshmallow==3.21.1
argon2-cffi==23.1.0
psycopg[binary]==3.1.18
python-dotenv==1.0.1
gunicorn==21.2.0
```

```bash
pip install -r requirements.txt
```

### 3.4 `.env` + Config

```bash
# .env (gitignore)
FLASK_ENV=development
SECRET_KEY=dev-secret-change-in-prod
JWT_SECRET_KEY=dev-jwt-secret-change-in-prod
DATABASE_URL=postgresql+psycopg://memo:memo@localhost:5432/memo
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

    JWT_SECRET_KEY = os.environ["JWT_SECRET_KEY"]
    JWT_ACCESS_TOKEN_EXPIRES = timedelta(minutes=15)
    JWT_REFRESH_TOKEN_EXPIRES = timedelta(days=14)

    CORS_ORIGINS = os.environ.get("CORS_ORIGINS", "http://localhost:3000").split(",")
```

> **`load_dotenv()`**: `.env` 파일을 `os.environ` 으로 로드. 운영에선 `.env` 대신 시스템 환경변수.

### 3.5 Extensions — SQLAlchemy / JWTManager 객체 한 곳에

```python
# app/extensions.py
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_jwt_extended import JWTManager
from flask_cors import CORS

db = SQLAlchemy()
migrate = Migrate()
jwt = JWTManager()
cors = CORS()
```

> **왜 분리**: `create_app` 안에 다 두면 import 순환 위험. 객체는 미리 만들고, `init_app(app)` 으로 나중에 연결.

### 3.6 App factory

```python
# app/__init__.py
from flask import Flask

from .config import Config
from .extensions import db, migrate, jwt, cors


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

    # Blueprint 등록
    from .api.auth import auth_bp
    from .api.memos import memos_bp
    app.register_blueprint(auth_bp)
    app.register_blueprint(memos_bp)

    # 에러 핸들러
    from .errors import register_error_handlers
    register_error_handlers(app)

    return app
```

> **`create_app` 패턴**: testing / production / staging 마다 다른 Config 주입 가능. Flask 의 표준 패턴.

### 3.7 entry point — `wsgi.py`

```python
# wsgi.py (project root)
from app import create_app

app = create_app()

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
```

```bash
# 실행
python wsgi.py
# 또는 flask CLI
export FLASK_APP=wsgi.py
flask run --debug
```

---

## 4. Domain + 데이터 model — SQLAlchemy 2.x

### 4.1 PostgreSQL 준비

**docker** 가 가장 빠름:

```bash
docker run -d \
  --name memo-db \
  -p 5432:5432 \
  -e POSTGRES_USER=memo \
  -e POSTGRES_PASSWORD=memo \
  -e POSTGRES_DB=memo \
  postgres:16
```

**brew (macOS)**:
```bash
brew install postgresql@16
brew services start postgresql@16
createuser -s memo
createdb -O memo memo
```

연결 테스트:
```bash
psql postgresql://memo:memo@localhost:5432/memo -c "SELECT 1"
# ?column?
# ----------
#         1
```

### 4.2 User / Memo 모델

```python
# app/domain/user.py
from datetime import datetime, timezone
from sqlalchemy import Column, Integer, String, DateTime
from sqlalchemy.orm import relationship

from app.extensions import db


class User(db.Model):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, autoincrement=True)
    email = Column(String(255), nullable=False, unique=True, index=True)
    password_hash = Column(String(255), nullable=False)
    name = Column(String(100), nullable=False)
    created_at = Column(
        DateTime(timezone=True),
        nullable=False,
        default=lambda: datetime.now(timezone.utc),
    )

    memos = relationship("Memo", back_populates="owner", cascade="all, delete-orphan")

    def __repr__(self) -> str:
        return f"<User id={self.id} email={self.email!r}>"
```

```python
# app/domain/memo.py
from datetime import datetime, timezone
from sqlalchemy import Column, Integer, String, Text, DateTime, ForeignKey
from sqlalchemy.orm import relationship

from app.extensions import db


class Memo(db.Model):
    __tablename__ = "memos"

    id = Column(Integer, primary_key=True, autoincrement=True)
    owner_id = Column(Integer, ForeignKey("users.id"), nullable=False, index=True)
    title = Column(String(200), nullable=False)
    content = Column(Text, nullable=False, default="")
    created_at = Column(DateTime(timezone=True), nullable=False, default=lambda: datetime.now(timezone.utc))
    updated_at = Column(
        DateTime(timezone=True),
        nullable=False,
        default=lambda: datetime.now(timezone.utc),
        onupdate=lambda: datetime.now(timezone.utc),
    )

    owner = relationship("User", back_populates="memos")
```

> **`Column(Integer, primary_key=True, autoincrement=True)`** — PG 의 SERIAL.
> **`relationship + ForeignKey`** — 1:N (User → Memo).
> **`__repr__`** — debug log 에 객체 식별 (없으면 `<User object at 0x...>`).

### 4.3 모델 import — `app/__init__.py` 에서

Flask-Migrate 가 모델을 알도록 `create_app` 안에서 import:

```python
# app/__init__.py 의 create_app() 끝부분
def create_app(config_class=Config) -> Flask:
    # ... (앞서 만든 코드)

    # 모델 import — Migrate 가 인지하도록
    from .domain import user, memo  # noqa: F401

    return app
```

`# noqa: F401` — "import 했지만 사용 안 함" lint 경고 무시 (의도적 side effect).

### 4.4 첫 마이그레이션 — Flask-Migrate (Alembic wrapper)

```bash
export FLASK_APP=wsgi.py

# 마이그레이션 디렉터리 초기화 (한 번만)
flask db init

# 모델 변경 → 마이그레이션 파일 생성
flask db migrate -m "create users and memos tables"
# migrations/versions/xxxx_create_users_and_memos_tables.py 생성됨

# 실제 DB 에 적용
flask db upgrade
```

확인:
```bash
psql postgresql://memo:memo@localhost:5432/memo -c "\dt"
# Schema |       Name        | Type  | Owner
# -------+-------------------+-------+-------
# public | alembic_version   | table | memo
# public | memos             | table | memo
# public | users             | table | memo
```

> **운영 절대 규칙**: `db.create_all()` 사용 X. 항상 Flask-Migrate 의 `migrate` + `upgrade`. Spring 의 Flyway 와 동등. ([[database/flyway]] 참고)

---

## 5. 입력 검증 — marshmallow

### 5.1 Schema 정의

```python
# app/schemas/auth_schema.py
from marshmallow import Schema, fields, validate


class SignupRequestSchema(Schema):
    email = fields.Email(required=True)
    password = fields.String(required=True, validate=validate.Length(min=8, max=64))
    name = fields.String(required=True, validate=validate.Length(min=1, max=100))


class LoginRequestSchema(Schema):
    email = fields.Email(required=True)
    password = fields.String(required=True)
```

```python
# app/schemas/memo_schema.py
from marshmallow import Schema, fields, validate


class CreateMemoSchema(Schema):
    title = fields.String(required=True, validate=validate.Length(min=1, max=200))
    content = fields.String(load_default="", validate=validate.Length(max=10_000))


class UpdateMemoSchema(Schema):
    title = fields.String(validate=validate.Length(min=1, max=200))
    content = fields.String(validate=validate.Length(max=10_000))


class MemoResponseSchema(Schema):
    id = fields.Integer()
    title = fields.String()
    content = fields.String()
    createdAt = fields.DateTime(attribute="created_at")
    updatedAt = fields.DateTime(attribute="updated_at")
```

> **`load_default`** — 입력에서 빠지면 default. **`attribute`** — model field name 과 JSON field name 다를 때.

### 5.2 사용

```python
from app.schemas.memo_schema import CreateMemoSchema

payload = CreateMemoSchema().load(request.get_json())
# 검증 실패 시 marshmallow.ValidationError 자동 raise
# payload = {"title": "...", "content": "..."}
```

---

## 6. 에러 처리 — Custom Exception + errorhandler

### 6.1 custom Exception

```python
# app/errors.py
import logging
from typing import Any
from flask import Flask, jsonify, request
from werkzeug.exceptions import HTTPException
from marshmallow import ValidationError

log = logging.getLogger(__name__)


class BusinessException(Exception):
    """도메인 / 비즈니스 예외 베이스."""

    def __init__(self, status: int, code: str, message: str) -> None:
        self.status = status
        self.code = code
        self.message = message
        super().__init__(message)


# 자주 쓰는 사전 정의
class NotFoundException(BusinessException):
    def __init__(self, what: str = "리소스") -> None:
        super().__init__(404, "NOT_FOUND", f"{what}를 찾을 수 없습니다.")


class ForbiddenException(BusinessException):
    def __init__(self, message: str = "권한이 없습니다.") -> None:
        super().__init__(403, "FORBIDDEN", message)


class UnauthorizedException(BusinessException):
    def __init__(self, message: str = "인증이 필요합니다.") -> None:
        super().__init__(401, "UNAUTHORIZED", message)


class ConflictException(BusinessException):
    def __init__(self, message: str = "이미 존재합니다.") -> None:
        super().__init__(409, "CONFLICT", message)


def _error_body(status: int, code: str, message: str, details: Any = None) -> dict:
    body = {"success": False, "error": {"code": code, "message": message}}
    if details is not None:
        body["error"]["details"] = details
    return body


def register_error_handlers(app: Flask) -> None:

    @app.errorhandler(BusinessException)
    def handle_business(ex: BusinessException):
        if ex.status >= 500:
            log.error("BIZ 5xx: %s", ex, exc_info=True)
        else:
            log.warning("BIZ %s: %s", ex.code, ex.message)
        return jsonify(_error_body(ex.status, ex.code, ex.message)), ex.status

    @app.errorhandler(ValidationError)
    def handle_validation(ex: ValidationError):
        log.warning("VALIDATION: %s", ex.messages)
        return jsonify(_error_body(422, "VALIDATION_ERROR",
                                   "입력 형식이 올바르지 않습니다.", ex.messages)), 422

    @app.errorhandler(HTTPException)
    def handle_http(ex: HTTPException):
        # 404 / 405 / 415 등 werkzeug 표준 에러
        return jsonify(_error_body(ex.code or 500, "HTTP_ERROR", ex.description or "")), ex.code or 500

    @app.errorhandler(Exception)
    def handle_any(ex: Exception):
        log.exception("UNCAUGHT")
        return jsonify(_error_body(500, "INTERNAL_SERVER_ERROR",
                                   "서버 내부 오류가 발생했습니다.")), 500
```

### 6.2 응답 헬퍼

```python
# app/responses.py
from flask import jsonify


def ok(data=None, message: str = "성공", status: int = 200):
    body = {"success": True, "message": message}
    if data is not None:
        body["data"] = data
    return jsonify(body), status
```

---

## 7. 인증 — Argon2 비밀번호 + JWT

### 7.1 비밀번호 해시

```python
# app/services/password.py
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError

_hasher = PasswordHasher()


def hash_password(plain: str) -> str:
    return _hasher.hash(plain)


def verify_password(plain: str, hashed: str) -> bool:
    try:
        _hasher.verify(hashed, plain)
        return True
    except VerifyMismatchError:
        return False
```

### 7.2 auth service

```python
# app/services/auth_service.py
from flask_jwt_extended import create_access_token, create_refresh_token

from app.domain.user import User
from app.extensions import db
from app.errors import ConflictException, UnauthorizedException
from .password import hash_password, verify_password


def signup(email: str, password: str, name: str) -> User:
    existing = db.session.query(User).filter(User.email == email).first()
    if existing is not None:
        raise ConflictException("이미 가입된 이메일입니다.")

    user = User(
        email=email,
        password_hash=hash_password(password),
        name=name,
    )
    db.session.add(user)
    db.session.commit()
    return user


def login(email: str, password: str) -> dict:
    user = db.session.query(User).filter(User.email == email).first()
    if user is None or not verify_password(password, user.password_hash):
        raise UnauthorizedException("이메일 또는 비밀번호가 올바르지 않습니다.")

    access_token = create_access_token(
        identity=str(user.id),
        additional_claims={"email": user.email, "name": user.name},
    )
    refresh_token = create_refresh_token(identity=str(user.id))
    return {
        "accessToken": access_token,
        "refreshToken": refresh_token,
        "tokenType": "Bearer",
        "expiresIn": 900,
    }
```

### 7.3 auth blueprint (endpoint)

```python
# app/api/auth.py
from flask import Blueprint, request

from app.schemas.auth_schema import SignupRequestSchema, LoginRequestSchema
from app.services import auth_service
from app.responses import ok

auth_bp = Blueprint("auth", __name__, url_prefix="/api/v1/auth")


@auth_bp.post("/signup")
def signup():
    payload = SignupRequestSchema().load(request.get_json())
    user = auth_service.signup(payload["email"], payload["password"], payload["name"])
    return ok(
        {"id": user.id, "email": user.email, "name": user.name},
        message="회원가입 성공",
        status=201,
    )


@auth_bp.post("/login")
def login():
    payload = LoginRequestSchema().load(request.get_json())
    tokens = auth_service.login(payload["email"], payload["password"])
    return ok(tokens, message="로그인 성공")
```

### 7.4 JWT loader (envelope 통일)

```python
# app/__init__.py 의 create_app 안에서
from flask import jsonify
from .extensions import jwt


def _register_jwt_loaders():
    @jwt.unauthorized_loader
    def _missing(_reason):
        return jsonify({"success": False, "error": {"code": "UNAUTHORIZED", "message": "인증이 필요합니다."}}), 401

    @jwt.invalid_token_loader
    def _invalid(_reason):
        return jsonify({"success": False, "error": {"code": "INVALID_TOKEN", "message": "유효하지 않은 토큰입니다."}}), 401

    @jwt.expired_token_loader
    def _expired(_jh, _jp):
        return jsonify({"success": False, "error": {"code": "INVALID_TOKEN", "message": "만료된 토큰입니다."}}), 401


# create_app 끝에서 호출
_register_jwt_loaders()
```

---

## 8. CRUD endpoint — 메모

### 8.1 service

```python
# app/services/memo_service.py
from app.domain.memo import Memo
from app.extensions import db
from app.errors import NotFoundException, ForbiddenException


def create(owner_id: int, title: str, content: str) -> Memo:
    memo = Memo(owner_id=owner_id, title=title, content=content)
    db.session.add(memo)
    db.session.commit()
    return memo


def list_for_owner(owner_id: int, limit: int = 20, offset: int = 0) -> list[Memo]:
    return (
        db.session.query(Memo)
        .filter(Memo.owner_id == owner_id)
        .order_by(Memo.created_at.desc())
        .limit(limit)
        .offset(offset)
        .all()
    )


def get_owned(memo_id: int, owner_id: int) -> Memo:
    memo = db.session.get(Memo, memo_id)
    if memo is None:
        raise NotFoundException("메모")
    if memo.owner_id != owner_id:
        raise ForbiddenException("본인의 메모만 조회 가능합니다.")
    return memo


def update(memo_id: int, owner_id: int, patch: dict) -> Memo:
    memo = get_owned(memo_id, owner_id)
    if "title" in patch:
        memo.title = patch["title"]
    if "content" in patch:
        memo.content = patch["content"]
    db.session.commit()
    return memo


def delete(memo_id: int, owner_id: int) -> None:
    memo = get_owned(memo_id, owner_id)
    db.session.delete(memo)
    db.session.commit()
```

### 8.2 blueprint

```python
# app/api/memos.py
from flask import Blueprint, request
from flask_jwt_extended import jwt_required, get_jwt_identity

from app.schemas.memo_schema import CreateMemoSchema, UpdateMemoSchema, MemoResponseSchema
from app.services import memo_service
from app.responses import ok

memos_bp = Blueprint("memos", __name__, url_prefix="/api/v1/memos")


@memos_bp.get("")
@jwt_required()
def list_memos():
    owner_id = int(get_jwt_identity())
    limit = min(int(request.args.get("limit", 20)), 100)
    offset = max(int(request.args.get("offset", 0)), 0)

    memos = memo_service.list_for_owner(owner_id, limit, offset)
    return ok(MemoResponseSchema(many=True).dump(memos), message="조회 성공")


@memos_bp.post("")
@jwt_required()
def create_memo():
    owner_id = int(get_jwt_identity())
    payload = CreateMemoSchema().load(request.get_json())
    memo = memo_service.create(owner_id, payload["title"], payload["content"])
    return ok(MemoResponseSchema().dump(memo), message="생성 완료", status=201)


@memos_bp.get("/<int:memo_id>")
@jwt_required()
def get_memo(memo_id: int):
    owner_id = int(get_jwt_identity())
    memo = memo_service.get_owned(memo_id, owner_id)
    return ok(MemoResponseSchema().dump(memo))


@memos_bp.patch("/<int:memo_id>")
@jwt_required()
def patch_memo(memo_id: int):
    owner_id = int(get_jwt_identity())
    payload = UpdateMemoSchema().load(request.get_json())
    memo = memo_service.update(memo_id, owner_id, payload)
    return ok(MemoResponseSchema().dump(memo), message="수정 완료")


@memos_bp.delete("/<int:memo_id>")
@jwt_required()
def delete_memo(memo_id: int):
    owner_id = int(get_jwt_identity())
    memo_service.delete(memo_id, owner_id)
    return ok(message="삭제 완료")
```

### 8.3 실행 + 직접 호출

```bash
flask run --debug
```

```bash
# 1. signup
curl -X POST http://localhost:5000/api/v1/auth/signup \
  -H "Content-Type: application/json" \
  -d '{"email":"yuchan@example.com","password":"P@ssw0rd1!","name":"yuchan"}'

# 2. login → token
TOKEN=$(curl -sX POST http://localhost:5000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"yuchan@example.com","password":"P@ssw0rd1!"}' \
  | jq -r .data.accessToken)

echo $TOKEN

# 3. 메모 생성
curl -X POST http://localhost:5000/api/v1/memos \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"첫 메모","content":"안녕하세요"}'

# 4. 목록
curl http://localhost:5000/api/v1/memos -H "Authorization: Bearer $TOKEN"

# 5. 수정
curl -X PATCH http://localhost:5000/api/v1/memos/1 \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"수정된 제목"}'

# 6. 삭제
curl -X DELETE http://localhost:5000/api/v1/memos/1 \
  -H "Authorization: Bearer $TOKEN"
```

---

## 9. 테스트 — pytest

### 9.1 의존성 + fixture

```bash
pip install pytest pytest-flask
```

```python
# tests/conftest.py
import pytest
from app import create_app
from app.config import Config
from app.extensions import db


class TestConfig(Config):
    TESTING = True
    SQLALCHEMY_DATABASE_URI = "sqlite:///:memory:"     # 가벼운 in-memory


@pytest.fixture
def app():
    app = create_app(TestConfig)
    with app.app_context():
        db.create_all()
        yield app
        db.drop_all()


@pytest.fixture
def client(app):
    return app.test_client()


@pytest.fixture
def auth_headers(client):
    """signup + login → Authorization 헤더."""
    client.post("/api/v1/auth/signup", json={
        "email": "test@example.com",
        "password": "P@ssw0rd1!",
        "name": "tester",
    })
    res = client.post("/api/v1/auth/login", json={
        "email": "test@example.com",
        "password": "P@ssw0rd1!",
    })
    token = res.get_json()["data"]["accessToken"]
    return {"Authorization": f"Bearer {token}"}
```

### 9.2 첫 테스트

```python
# tests/test_memos.py
def test_create_and_list_memo(client, auth_headers):
    # 생성
    res = client.post("/api/v1/memos",
                     json={"title": "t1", "content": "c1"},
                     headers=auth_headers)
    assert res.status_code == 201
    memo_id = res.get_json()["data"]["id"]

    # 목록
    res = client.get("/api/v1/memos", headers=auth_headers)
    assert res.status_code == 200
    items = res.get_json()["data"]
    assert len(items) == 1
    assert items[0]["id"] == memo_id
    assert items[0]["title"] == "t1"


def test_cannot_access_others_memo(client, auth_headers):
    # 다른 user 만들고 memo 생성
    client.post("/api/v1/auth/signup", json={
        "email": "other@example.com",
        "password": "P@ssw0rd1!",
        "name": "other",
    })
    res = client.post("/api/v1/auth/login", json={
        "email": "other@example.com",
        "password": "P@ssw0rd1!",
    })
    other_token = res.get_json()["data"]["accessToken"]
    other_headers = {"Authorization": f"Bearer {other_token}"}

    res = client.post("/api/v1/memos",
                     json={"title": "secret", "content": "..."},
                     headers=other_headers)
    other_memo_id = res.get_json()["data"]["id"]

    # 본인이 아닌 user 가 접근 → 403
    res = client.get(f"/api/v1/memos/{other_memo_id}", headers=auth_headers)
    assert res.status_code == 403
    assert res.get_json()["error"]["code"] == "FORBIDDEN"


def test_validation_error(client, auth_headers):
    res = client.post("/api/v1/memos",
                     json={"content": "title 누락"},
                     headers=auth_headers)
    assert res.status_code == 422
    body = res.get_json()
    assert body["error"]["code"] == "VALIDATION_ERROR"
    assert "title" in body["error"]["details"]
```

```bash
pytest -v
# ===== test session starts =====
# tests/test_memos.py::test_create_and_list_memo PASSED
# tests/test_memos.py::test_cannot_access_others_memo PASSED
# tests/test_memos.py::test_validation_error PASSED
# ===== 3 passed in 0.5s =====
```

---

## 10. 배포 — gunicorn + Docker

### 10.1 gunicorn (production WSGI server)

Flask 의 dev server (`app.run`) 는 single-thread / debug 용. **운영은 gunicorn**.

```bash
pip install gunicorn
```

```bash
gunicorn -w 4 -b 0.0.0.0:5000 "wsgi:app"
# -w 4   : worker process 4개
# -b ... : bind
# wsgi:app : module:variable
```

> **worker 수 시작점**: `(CPU * 2) + 1`. EC2 c5.large (2 vCPU) → 5 workers.

### 10.2 Dockerfile

```dockerfile
# Dockerfile
FROM python:3.12-slim

# 운영 의존성 설치 (build tool 등 cache 효율)
RUN apt-get update && apt-get install -y --no-install-recommends \
        libpq5 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# requirements 먼저 → layer cache
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 소스 복사
COPY app/ ./app/
COPY wsgi.py .
COPY migrations/ ./migrations/

# 비root user (보안)
RUN useradd -m -u 1000 flask && chown -R flask:flask /app
USER flask

ENV PORT=5000
EXPOSE 5000

# entrypoint
CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:5000", "--access-logfile", "-", "wsgi:app"]
```

### 10.3 docker-compose (개발)

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: memo
      POSTGRES_PASSWORD: memo
      POSTGRES_DB: memo
    ports: ["5432:5432"]
    volumes: ["dbdata:/var/lib/postgresql/data"]

  app:
    build: .
    environment:
      SECRET_KEY: dev-secret
      JWT_SECRET_KEY: dev-jwt
      DATABASE_URL: postgresql+psycopg://memo:memo@db:5432/memo
    ports: ["5000:5000"]
    depends_on: [db]
    command: >
      sh -c "flask db upgrade &&
             gunicorn -w 4 -b 0.0.0.0:5000 --access-logfile - wsgi:app"

volumes:
  dbdata:
```

```bash
docker compose up --build
```

### 10.4 운영 체크리스트

- [ ] `DEBUG=False` (default Flask 3.x — `app.debug` 가 environment 로 결정)
- [ ] `SECRET_KEY` / `JWT_SECRET_KEY` 가 vault / Secrets Manager (env 직접 X)
- [ ] HTTPS (Nginx / ALB 가 TLS termination)
- [ ] `gunicorn --workers` 가 인스턴스 spec 에 맞게
- [ ] `gunicorn --timeout` 명시 (default 30s — 큰 요청 처리 시 조정)
- [ ] PostgreSQL `max_connections` 와 workers × pool_size 매칭
- [ ] `flask db upgrade` 가 컨테이너 startup 에 자동
- [ ] log 가 stdout/stderr (Docker / Kubernetes 가 수집)
- [ ] healthcheck endpoint (`/healthz` 같은 1줄 endpoint)

```python
# app/api/health.py (선택)
from flask import Blueprint
health_bp = Blueprint("health", __name__)

@health_bp.get("/healthz")
def healthz():
    return {"status": "ok"}, 200
```

→ ALB / Kubernetes liveness/readiness probe 용.

---

## 11. 자주 막히는 함정

### 함정 1 — `app.app_context()` 안에서만 `db.session` 동작
```python
# ❌ context 밖
print(db.session.query(User).count())     # RuntimeError: Working outside of application context

# ✅ context 안
with app.app_context():
    print(db.session.query(User).count())
```

CLI script / 테스트 / Celery task 에서 자주 만남. **`app.app_context()` 명시**.

### 함정 2 — circular import
`app/extensions.py` 와 `app/domain/user.py` 가 서로 import 시도 → ImportError.
**`extensions.py` 에 db / jwt 객체만 둠, model 은 도메인 패키지에**. import 트리 단방향.

### 함정 3 — Flask debug mode = `WERKZEUG_DEBUG_PIN`
`debug=True` 면 콘솔에 PIN 출력 → 누구나 코드 실행 가능. **운영 절대 False**.

### 함정 4 — `request.get_json()` 가 None
Content-Type 가 `application/json` 아니면 None 반환. **클라가 헤더 보내는지 확인** 또는 `force=True`.

### 함정 5 — Flask-SQLAlchemy 2.x vs 3.x
3.x 는 declarative_base 자동 + Query 인터페이스 변경. 본 가이드는 **3.x 기준**. 옛 튜토리얼 (`User.query.filter(...)`) 는 deprecated — `db.session.query(User).filter(...)` 권장.

### 함정 6 — gunicorn worker timeout
긴 요청 (PDF 생성 등) 이 30s 넘으면 worker kill + restart. **`--timeout 120`** 또는 Celery 로 분리.

### 함정 7 — PostgreSQL 연결 폭증
worker 4 × pool 10 = 인스턴스당 40 connection. PG `max_connections=100` 면 multi-instance 어려움. **PgBouncer** 또는 pool 줄이기.

### 함정 8 — `os.environ["X"]` KeyError 시 컨테이너 crash loop
docker 가 무한 restart → kubectl logs 만으론 원인 어려움. **`os.environ.get("X", default)`** + startup check.

### 함정 9 — 마이그레이션 conflict (여러 dev 가 동시 작성)
같은 parent 를 가진 V_a, V_b 동시 생성 → upgrade 시 충돌. **PR review 시 단일 마이그레이션 chain 유지**, `flask db merge` 로 합치기.

### 함정 10 — JWT 발급 시 `identity` 가 str 아닌 int
`create_access_token(identity=user.id)` 에서 `user.id` 가 int → 직렬화 OK 지만, `get_jwt_identity()` 가 str/int 어느 타입으로 돌아올지 모호. **`str(user.id)`** 통일 권장.

---

## 12. 다음 학습 로드맵

| 단계 | 학습할 것 | vault 노트 |
| --- | --- | --- |
| 1 | 본 가이드 직접 따라 만들기 | (이 노트) |
| 2 | 응답 envelope 통일 | [[common/response-envelope]] |
| 3 | 더 강한 보안 (CORS / JWT loader / Argon2 옵션) | [[common/security-config]] |
| 4 | SQLAlchemy 깊이 + N+1 회피 | [[database/database]] |
| 5 | Redis 캐시 / rate-limit / distributed lock | [[api-design/cache-redis]] / [[api-design/rate-limiting]] / [[api-design/distributed-lock]] |
| 6 | 파일 업로드 (S3) | [[api-design/file-upload-s3]] |
| 7 | 비동기 작업 (Celery) | [[api-design/webhook-send]] / [[api-design/recommendation]] |
| 8 | WebSocket | [[api-design/websocket-stomp]] |
| 9 | 운영 / 모니터링 | (예정 — Prometheus / Sentry) |

→ 본 가이드만으로도 작은 SaaS / 사이드 프로젝트는 충분. 위 노트는 production 운영 시 점진적으로.

---

## 13. 참고 / cross-link

- [[python-flask|↑ Flask area]]
- [[../python-fastapi/getting-started]] (예정 — FastAPI 변형)
- [[../python-django/getting-started]] (예정 — Django 변형)
- [[common/response-envelope]] — 응답 envelope 표준
- [[common/security-config]] — JWT + CORS 표준
- [[api-design/api-design]] — 실전 레시피 hub
- [[database/database]] — SQLAlchemy / ORM
- Flask 공식 docs: <https://flask.palletsprojects.com/>
- Flask-SQLAlchemy: <https://flask-sqlalchemy.palletsprojects.com/>
- Flask-JWT-Extended: <https://flask-jwt-extended.readthedocs.io/>
- marshmallow: <https://marshmallow.readthedocs.io/>
- SQLAlchemy 2.x: <https://docs.sqlalchemy.org/en/20/>
