---
title: "05. Flask-JWT-Extended + Argon2 + gunicorn + Docker"
kind: guide
status: current
created_at: 2026-05-27T13:00:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-flask, guidelines, security, jwt, deployment, docker]
---

# 05. Flask-JWT-Extended + Argon2 + gunicorn + Docker

**[[guidelines|↑ Hub]]** ← [[04-controller-and-validation]] · → 다음 [[06-swagger-openapi]]

> Flask-JWT-Extended (JWT 발급/검증) + Argon2 (비밀번호 해시) + gunicorn + Docker 배포.

---

## 1. 의존성

```bash
pip install Flask-JWT-Extended Flask-CORS argon2-cffi gunicorn
```

---

## 2. JWT 설정

```python
# app/config.py — 이미 [[02-project-structure]] 에서
class Config:
    JWT_SECRET_KEY = os.environ["JWT_SECRET_KEY"]
    JWT_ACCESS_TOKEN_EXPIRES = timedelta(minutes=15)
    JWT_REFRESH_TOKEN_EXPIRES = timedelta(days=14)
    JWT_TOKEN_LOCATION = ["headers"]
    JWT_HEADER_NAME = "Authorization"
    JWT_HEADER_TYPE = "Bearer"
    JWT_ALGORITHM = "HS256"
```

---

## 3. JWT loader — envelope 통일

```python
# app/common/jwt_loaders.py
from flask import jsonify
from flask_jwt_extended import JWTManager


def register_jwt_loaders(jwt: JWTManager) -> None:

    @jwt.unauthorized_loader
    def _missing(_reason: str):
        return jsonify({
            "success": False,
            "error": {"code": "UNAUTHORIZED", "message": "인증이 필요합니다."},
        }), 401

    @jwt.invalid_token_loader
    def _invalid(_reason: str):
        return jsonify({
            "success": False,
            "error": {"code": "INVALID_TOKEN", "message": "유효하지 않은 토큰입니다."},
        }), 401

    @jwt.expired_token_loader
    def _expired(_jh, _jp):
        return jsonify({
            "success": False,
            "error": {"code": "INVALID_TOKEN", "message": "만료된 토큰입니다."},
        }), 401

    @jwt.needs_fresh_token_loader
    def _fresh(_jh, _jp):
        return jsonify({
            "success": False,
            "error": {"code": "UNAUTHORIZED", "message": "fresh 토큰 필요"},
        }), 401
```

`create_app(...)` 안:

```python
from .common.jwt_loaders import register_jwt_loaders
register_jwt_loaders(jwt)
```

→ 모든 401 응답이 같은 envelope.

---

## 4. 비밀번호 해시 — Argon2

```python
# app/services/password.py
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError

_hasher = PasswordHasher(
    time_cost=3,                  # iteration
    memory_cost=64 * 1024,        # 64 MB
    parallelism=2,
    hash_len=32,
    salt_len=16,
)


def hash_password(plain: str) -> str:
    return _hasher.hash(plain)


def verify_password(plain: str, hashed: str) -> bool:
    try:
        _hasher.verify(hashed, plain)
        return True
    except VerifyMismatchError:
        return False


def needs_rehash(hashed: str) -> bool:
    return _hasher.check_needs_rehash(hashed)
```

> **Argon2id** = OWASP 권장. bcrypt cost 12+ 도 OK.

---

## 5. Auth service — signup / login

```python
# app/services/auth_service.py
from flask_jwt_extended import create_access_token, create_refresh_token

from app.common.exceptions import ConflictException, UnauthorizedException
from app.domain.user import User
from app.extensions import db
from .password import hash_password, verify_password


def signup(email: str, password: str, name: str) -> User:
    existing = db.session.scalar(
        db.session.query(User).filter(User.email == email).statement
    )
    if existing is not None:
        raise ConflictException("이미 가입된 이메일입니다.")

    user = User(
        email=email,
        password_hash=hash_password(password),
        name=name,
    )
    db.session.add(user)
    db.session.commit()
    db.session.refresh(user)
    return user


def login(email: str, password: str) -> dict:
    from sqlalchemy import select
    user = db.session.scalar(select(User).where(User.email == email))
    if user is None or not verify_password(password, user.password_hash):
        raise UnauthorizedException("이메일 또는 비밀번호가 올바르지 않습니다.")

    access = create_access_token(
        identity=str(user.id),                  # ⚠️ str — 함정 §10 참고
        additional_claims={"email": user.email, "name": user.name, "role": "USER"},
    )
    refresh = create_refresh_token(identity=str(user.id))

    return {
        "accessToken": access,
        "refreshToken": refresh,
        "tokenType": "Bearer",
        "expiresIn": 900,
    }
```

---

## 6. Auth blueprint

```python
# app/api/auth.py
from flask import Blueprint, request

from app.common.responses import ok
from app.schemas.auth import SignupRequestSchema, LoginRequestSchema
from app.services import auth_service

auth_bp = Blueprint("auth", __name__, url_prefix="/api/v1/auth")


@auth_bp.post("/signup")
def signup():
    payload = SignupRequestSchema().load(request.get_json())
    user = auth_service.signup(payload["email"], payload["password"], payload["name"])
    return ok(
        {"id": user.id, "email": user.email, "name": user.name},
        "회원가입 성공",
        status=201,
    )


@auth_bp.post("/login")
def login():
    payload = LoginRequestSchema().load(request.get_json())
    tokens = auth_service.login(payload["email"], payload["password"])
    return ok(tokens, "로그인 성공")
```

```python
# app/schemas/auth.py
from marshmallow import Schema, fields, validate


class SignupRequestSchema(Schema):
    email = fields.Email(required=True)
    password = fields.String(required=True, validate=validate.Length(min=8, max=64))
    name = fields.String(required=True, validate=validate.Length(min=1, max=100))


class LoginRequestSchema(Schema):
    email = fields.Email(required=True)
    password = fields.String(required=True)
```

---

## 7. CORS — Flask-CORS

`create_app(...)` 안에서 이미 설정 ([[02-project-structure#6 App factory|02 §6]]):

```python
cors.init_app(
    app,
    resources={r"/api/*": {"origins": app.config["CORS_ORIGINS"]}},
    supports_credentials=False,                       # JWT + Bearer 면 false
    methods=["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
    allow_headers=["Authorization", "Content-Type", "Accept-Language"],
)
```

---

## 8. 권한 — role-based decorator

```python
# app/common/auth.py
from functools import wraps
from typing import Callable
from flask_jwt_extended import jwt_required, get_jwt

from app.common.exceptions import BusinessException


def role_required(*allowed: str) -> Callable:
    def decorator(fn):
        @wraps(fn)
        @jwt_required()
        def wrapper(*args, **kwargs):
            claims = get_jwt()
            role = claims.get("role", "")
            if role not in allowed:
                raise BusinessException(403, "INSUFFICIENT_PERMISSION",
                                        "해당 리소스에 접근할 권한이 없습니다.")
            return fn(*args, **kwargs)
        return wrapper
    return decorator
```

```python
@admin_bp.delete("/products/<int:id>")
@role_required("ADMIN", "MASTER")
def admin_delete(id: int):
    # ...
    return ok("삭제")
```

> 자세한 권한 모델 ([[../common/security-config]]) 참고.

---

## 9. 실습 — curl 전 흐름

```bash
# dev server
flask run --debug
# 또는
python wsgi.py

# 1. signup
curl -X POST http://localhost:5000/api/v1/auth/signup \
  -H "Content-Type: application/json" \
  -d '{"email":"yuchan@example.com","password":"P@ssw0rd1!","name":"yuchan"}'
# 201 + {"success":true,"data":{...},"message":"회원가입 성공"}

# 2. login
TOKEN=$(curl -sX POST http://localhost:5000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"yuchan@example.com","password":"P@ssw0rd1!"}' \
  | jq -r '.data.accessToken')

# 3. 메모 생성
curl -X POST http://localhost:5000/api/v1/memos \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"첫 메모","content":"안녕"}'

# 4. 토큰 없이 호출 → 401
curl http://localhost:5000/api/v1/memos
# 401 + {"success":false,"error":{"code":"UNAUTHORIZED",...}}
```

---

## 10. 배포 — gunicorn

Flask dev server (`app.run`) = single thread / dev only. 운영 = **gunicorn**.

```bash
gunicorn -w 4 -b 0.0.0.0:5000 "wsgi:app"
```

| 옵션 | 의미 |
| --- | --- |
| `-w N` | worker process 수 |
| `-b host:port` | bind |
| `-k uvicorn.workers.UvicornWorker` | async worker (Quart 등) |
| `--timeout 120` | 요청 timeout (default 30s) |
| `--access-logfile -` | access log → stdout |
| `--worker-class gevent` | gevent worker (sync IO 많을 때) |

> **worker 수**: `(CPU * 2) + 1`. 단 DB pool / 외부 IO 고려.

---

## 11. Dockerfile

```dockerfile
FROM python:3.12-slim

# PostgreSQL client (psycopg)
RUN apt-get update && apt-get install -y --no-install-recommends \
        libpq5 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# 의존성 먼저 (cache)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app/ ./app/
COPY wsgi.py .
COPY migrations/ ./migrations/

# 비root user (보안)
RUN useradd -m -u 1000 flask && chown -R flask:flask /app
USER flask

ENV PORT=5000 \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1
EXPOSE 5000

CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:5000", "--access-logfile", "-", "wsgi:app"]
```

> **`PYTHONUNBUFFERED=1`** = stdout 즉시 flush (Docker / k8s 가 로그 수집).

---

## 12. docker-compose

```yaml
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
      JWT_SECRET_KEY: dev-jwt-secret-32-bytes-please-32-bytes-please
      DATABASE_URL: postgresql+psycopg://memo:memo@db:5432/memo
      FLASK_APP: wsgi.py
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

---

## 13. 운영 체크리스트

- [ ] `DEBUG=False` (default 가 안전)
- [ ] `JWT_SECRET_KEY` 가 vault / Secrets Manager (env 직접 X)
- [ ] HTTPS (Nginx / ALB TLS termination)
- [ ] `gunicorn --workers` 인스턴스 spec 에 맞게
- [ ] `gunicorn --timeout` 명시
- [ ] PostgreSQL `max_connections` ≥ workers × pool_size
- [ ] `flask db upgrade` 가 container startup 자동
- [ ] log → stdout
- [ ] healthcheck endpoint (`/healthz` 별도)
- [ ] `JWT_SECRET_KEY` 32 bytes 이상
- [ ] Argon2 cost (time=3, memory=65536, parallelism=2)
- [ ] CORS `allowed-origins` 명시 (`*` X)

---

## 14. 자주 막히는 함정

### 함정 1 — `app.run()` 운영 사용
single thread / dev only. **gunicorn**.

### 함정 2 — `DEBUG=True` 운영
콘솔 PIN → 누구나 코드 실행. **운영 False**.

### 함정 3 — `JWT_SECRET_KEY` 가 짧음
HMAC-SHA256 약함. **32 bytes (256 bits) 이상** — `openssl rand -hex 32`.

### 함정 4 — `JWT_HEADER_TYPE` 누락
default `Bearer` 지만 명시 권장. 클라가 `Authorization: Bearer xxx` 안 보내면 401.

### 함정 5 — CORS `supports_credentials=True` + `origins="*"`
브라우저가 거부. JWT + Bearer 면 `False`.

### 함정 6 — jwt loader override 안 함
default 응답이 raw `{"msg": "..."}` — envelope 깨짐. **모든 loader 등록**.

### 함정 7 — Argon2 cost 너무 큼
운영 트래픽에 1초+ 걸림. 본 가이드 (3, 65536, 2) = ~100ms.

### 함정 8 — gunicorn worker timeout
긴 요청 (PDF 생성) 이 30s 넘으면 worker kill. **`--timeout 120`** 또는 Celery 분리.

### 함정 9 — `decorator` 안 `@wraps` 누락
Flask routing 이 wrapper 의 `__name__` 사용 → 같은 이름 두 endpoint 등록 conflict.

### 함정 10 — `create_access_token(identity=user.id)` 가 int
`get_jwt_identity()` 가 str/int 어느 type 인지 모호. **`str(user.id)` 통일**.

---

## 15. 다음

✅ Security + 배포 끝.

→ [[06-swagger-openapi]] — Flask-Smorest 또는 apispec — Swagger UI 자동
