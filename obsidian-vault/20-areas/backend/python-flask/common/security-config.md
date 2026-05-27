---
title: "SecurityConfig + JWT + CORS — 표준 세팅 (Flask)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T16:10:00+09:00
tags:
  - backend
  - python-flask
  - common
  - security
  - jwt
  - cors
  - production
---

# SecurityConfig + JWT + CORS — 표준 세팅 (Flask)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + Flask-JWT-Extended + Flask-CORS |

**[[../python-flask|↑ Flask]]** · **[[../../spring/java-spring/common/security-config|↗ Java 원본]]** · **[[../../spring/kotlin-spring/common/security-config|↗ Kotlin 변형]]**

> Flask 진영의 **JWT + CORS + 권한** 표준 세팅. 401/403 응답은 [[response-envelope]] 의 `CommonResponse` 통과.

---

## 1. 컴포넌트 한눈에

```
WSGI request
   │
   ├─ Flask-CORS (응답 헤더)
   ├─ before_request: JWT verify (Flask-JWT-Extended)
   │     └ 401 응답 시 CommonResponse envelope (JwtAuthenticationEntryPoint 동등)
   ├─ @jwt_required() / @permission_required("ROLE")
   │     └ 403 응답 시 CommonResponse envelope
   ├─ blueprint view
   └─ ApiLogMiddleware (응답 후 access log)

빈:
  PasswordHasher = argon2-cffi
  JWTManager (Flask-JWT-Extended)
  CORS (Flask-CORS)
```

---

## 2. 표준 의존성

```bash
# requirements.txt 또는 pyproject.toml
pip install flask>=3.0 \
    Flask-JWT-Extended>=4.6 \
    Flask-CORS>=4.0 \
    argon2-cffi>=23.1 \
    marshmallow>=3.20 \
    python-dotenv
```

```toml
# pyproject.toml (PEP 621)
[project]
dependencies = [
    "flask>=3.0",
    "Flask-JWT-Extended>=4.6",
    "Flask-CORS>=4.0",
    "argon2-cffi>=23.1",
    "marshmallow>=3.20",
]
```

---

## 3. App factory + JWT / CORS / errorhandler

```python
# app/__init__.py
from flask import Flask, jsonify
from flask_cors import CORS
from flask_jwt_extended import JWTManager

from .config import Config
from .common.error_handlers import register_error_handlers
from .common.logging_filter import ApiLogMiddleware
from .common.dto import CommonResponse
from .common.enums import ResponseCode


def create_app() -> Flask:
    app = Flask(__name__)
    app.config.from_object(Config)

    # ----- CORS -----
    CORS(
        app,
        resources={r"/api/*": {"origins": app.config["CORS_ORIGINS"]}},
        supports_credentials=False,                # JWT + Authorization 헤더면 False
        methods=["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
        allow_headers=["Authorization", "Content-Type", "Accept-Language", "X-DEVICE-UUID"],
        max_age=3600,
    )

    # ----- JWT -----
    jwt = JWTManager(app)
    _register_jwt_handlers(jwt)

    # ----- Global errorhandler -----
    register_error_handlers(app)

    # ----- ApiLog middleware -----
    app.wsgi_app = ApiLogMiddleware(app.wsgi_app)

    # blueprints
    from .api.auth import auth_bp
    from .api.users import users_bp
    app.register_blueprint(auth_bp)
    app.register_blueprint(users_bp)
    return app


def _register_jwt_handlers(jwt: JWTManager) -> None:
    """JWT 검증 실패 시 CommonResponse 형태로 응답 — Java 의 JwtAuthenticationEntryPoint 동등."""

    @jwt.unauthorized_loader
    def missing_token(reason: str):
        body = CommonResponse.fail(ResponseCode.UNAUTHORIZED, "인증이 필요합니다.")
        return jsonify(body.to_dict()), 401

    @jwt.invalid_token_loader
    def invalid_token(reason: str):
        body = CommonResponse.fail(ResponseCode.INVALID_TOKEN, "유효하지 않은 토큰입니다.")
        return jsonify(body.to_dict()), 401

    @jwt.expired_token_loader
    def expired_token(jwt_header, jwt_payload):
        body = CommonResponse.fail(ResponseCode.INVALID_TOKEN, "만료된 토큰입니다.")
        return jsonify(body.to_dict()), 401

    @jwt.needs_fresh_token_loader
    def needs_fresh(jwt_header, jwt_payload):
        body = CommonResponse.fail(ResponseCode.UNAUTHORIZED, "fresh 토큰이 필요합니다.")
        return jsonify(body.to_dict()), 401
```

---

## 4. Config

```python
# app/config.py
import os
from datetime import timedelta


class Config:
    # JWT
    JWT_SECRET_KEY = os.environ["JWT_SECRET_KEY"]     # base64 256-bit+ 시크릿
    JWT_ALGORITHM = "HS256"
    JWT_ACCESS_TOKEN_EXPIRES = timedelta(minutes=15)
    JWT_REFRESH_TOKEN_EXPIRES = timedelta(days=14)
    JWT_TOKEN_LOCATION = ["headers"]                  # cookie 아님
    JWT_HEADER_NAME = "Authorization"
    JWT_HEADER_TYPE = "Bearer"
    JWT_ISSUER = "shop.example.com"

    # CORS
    CORS_ORIGINS = os.environ.get("CORS_ORIGINS", "http://localhost:3000").split(",")
```

> **함정**: `JWT_SECRET_KEY` 가 32 bytes 미만이면 HMAC-SHA256 약함. **256-bit 이상 + vault 주입**.

---

## 5. JWT token 발급 (login 흐름)

```python
# app/api/auth.py
from datetime import datetime, timezone, timedelta
import uuid
from flask import Blueprint, request, jsonify
from flask_jwt_extended import (
    create_access_token, create_refresh_token, jwt_required, get_jwt_identity, get_jwt,
)

from app.common.exceptions import BusinessException
from app.common.enums import ResponseCode
from app.common.responses import ok

auth_bp = Blueprint("auth", __name__, url_prefix="/api/v1/auth")


@auth_bp.post("/login")
def login():
    payload = LoginRequestSchema().load(request.get_json())
    user = user_service.authenticate(payload["email"], payload["password"])

    if user is None:
        raise BusinessException(
            ResponseCode.UNAUTHORIZED,
            "입력하신 이메일과 비밀번호를 확인해 주세요.",
        )

    session_id = str(uuid.uuid4())
    additional_claims = {
        "role": user.role.name,
        "sid": session_id,
        "id": user.id,
    }
    access_token = create_access_token(
        identity=user.email,
        additional_claims=additional_claims,
    )
    refresh_token = create_refresh_token(identity=user.email)

    return ok(
        "로그인 성공",
        {
            "accessToken": access_token,
            "refreshToken": refresh_token,
            "tokenType": "Bearer",
            "expiresIn": 900,
        },
    )


@auth_bp.post("/token/refresh")
@jwt_required(refresh=True)
def refresh_token_endpoint():
    email = get_jwt_identity()
    user = user_service.find_active_by_email(email)
    if user is None:
        raise BusinessException(ResponseCode.NOT_FOUND, "사용자를 찾을 수 없습니다.")

    new_access = create_access_token(
        identity=email,
        additional_claims={"role": user.role.name, "sid": str(uuid.uuid4()), "id": user.id},
    )
    return ok("토큰 갱신", {"accessToken": new_access})
```

> **Flask-JWT-Extended idiom**:
> - `@jwt_required()` decorator — Spring `@PreAuthorize` 와 비슷.
> - `@jwt_required(refresh=True)` — refresh token 만 허용.
> - `additional_claims` — 표준 JWT claim 외 추가.

---

## 6. 권한 — custom decorator

```python
# app/common/auth.py
from functools import wraps
from typing import Callable
from flask_jwt_extended import jwt_required, get_jwt

from .exceptions import BusinessException
from .enums import ResponseCode


def role_required(*allowed_roles: str) -> Callable:
    """role 기반 권한 — Spring @PreAuthorize("hasAnyRole(...)") 동등."""
    def decorator(fn):
        @wraps(fn)
        @jwt_required()
        def wrapper(*args, **kwargs):
            claims = get_jwt()
            role = claims.get("role", "")
            if role not in allowed_roles:
                raise BusinessException(
                    ResponseCode.INSUFFICIENT_PERMISSION,
                    "해당 리소스에 접근할 권한이 없습니다.",
                )
            return fn(*args, **kwargs)
        return wrapper
    return decorator


def authenticated(fn):
    """jwt_required + claims 검증 wrapper — 본인 검증 등."""
    @wraps(fn)
    @jwt_required()
    def wrapper(*args, **kwargs):
        return fn(*args, **kwargs)
    return wrapper
```

```python
# 사용
from app.common.auth import role_required


@bp.post("/admin/products/<product_id>/hide")
@role_required("ADMIN", "MASTER")
def admin_hide_product(product_id: str):
    product_service.hide(product_id)
    return ok("숨김 처리")
```

> **Python idiom**: `@wraps(fn)` — decorator 가 원래 함수의 `__name__` / `__doc__` 보존. 자동 docs 생성 / Flask routing 정상.

---

## 7. ownership 검증

```python
# app/common/auth.py 에 추가
from flask import abort, g
from flask_jwt_extended import get_jwt


def owns_or_admin(check_fn: Callable[[dict, str], bool]):
    """check_fn(jwt_claims, resource_id) → bool. 본인 또는 admin 이면 통과."""
    def decorator(fn):
        @wraps(fn)
        @jwt_required()
        def wrapper(*args, **kwargs):
            claims = get_jwt()
            role = claims.get("role", "")
            resource_id = kwargs.get("resource_id") or args[0]

            if role in ("ADMIN", "MASTER"):
                return fn(*args, **kwargs)
            if check_fn(claims, resource_id):
                return fn(*args, **kwargs)
            raise BusinessException(ResponseCode.FORBIDDEN, "본인의 리소스만 접근 가능합니다.")
        return wrapper
    return decorator


# 사용 — product owner 검증
def _is_product_owner(claims: dict, product_id: str) -> bool:
    user_id = claims.get("id")
    product = product_repo.find_by_id(product_id)
    return product is not None and product.seller_id == user_id


@bp.patch("/products/<resource_id>")
@owns_or_admin(_is_product_owner)
def update_product(resource_id: str):
    # ...
    return ok("수정 완료")
```

---

## 8. 비밀번호 해시 — Argon2

```python
# app/common/password.py
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError

_hasher = PasswordHasher(
    time_cost=3,                 # iteration
    memory_cost=64 * 1024,       # 64 MB
    parallelism=2,
    hash_len=32,
    salt_len=16,
)


def hash_password(plain: str) -> str:
    return _hasher.hash(plain)


def verify_password(plain: str, hashed: str) -> bool:
    try:
        _hasher.verify(hashed, plain)
        # rehash 필요 시 — Argon2 cost 변경 등
        if _hasher.check_needs_rehash(hashed):
            # 호출자에서 new_hash = hash_password(plain) 후 DB 업데이트
            pass
        return True
    except VerifyMismatchError:
        return False
```

> **함정**: `bcrypt` 도 OK 지만 Argon2id (winner of password hashing competition) 권장. `passlib` 의 추상화도 가능.

---

## 9. yaml / env 설정

```bash
# .env (gitignore)
JWT_SECRET_KEY=base64encoded256bitkey...
CORS_ORIGINS=https://shop.example.com,https://admin.example.com
DB_URL=postgresql://user:pwd@localhost/shop
```

```python
# app/config.py 에 dotenv 로드
from dotenv import load_dotenv
load_dotenv()
```

---

## 10. 운영 체크리스트

- [ ] `JWT_SECRET_KEY` Base64 256-bit+, vault / .env (gitignore)
- [ ] `JWT_ACCESS_TOKEN_EXPIRES` 짧게 (15분)
- [ ] `JWT_REFRESH_TOKEN_EXPIRES` 14~30일
- [ ] CORS `origins` 명시적 (와일드카드 X)
- [ ] `supports_credentials=False` + Authorization 헤더 조합
- [ ] `jwt.unauthorized_loader` / `invalid_token_loader` / `expired_token_loader` 모두 envelope
- [ ] Argon2 또는 bcrypt cost 12+
- [ ] 모든 endpoint 에 `@jwt_required` 또는 `@role_required` (public 만 명시 제외)
- [ ] `ApiLogMiddleware` 가 모든 요청 기록
- [ ] WSGI worker (gunicorn) `--workers` 수 명시 + `--preload`

---

## 11. 함정 모음 (Flask + Python 한정)

### 함정 1 — `JWT_SECRET_KEY` 가 짧음
HMAC-SHA256 약함. `secrets.token_bytes(32)` 또는 vault 주입.

### 함정 2 — `JWT_HEADER_TYPE` 누락
default `Bearer` 지만 명시 권장. 클라가 `Authorization: Bearer xxx` 안 보내면 401.

### 함정 3 — CORS `supports_credentials=True` + `origins="*"`
브라우저가 거부. JWT + Authorization 헤더면 `False`.

### 함정 4 — `jwt.expired_token_loader` 미설정
default 응답이 plain JSON `{"msg": "Token has expired"}` — envelope 깨짐. 모든 loader override.

### 함정 5 — `before_request` 안 무거운 DB 조회
모든 요청에 부하. JWT decode 만 + 권한은 decorator 단.

### 함정 6 — Flask debug 모드를 운영에
`app.debug = True` 면 코드 / 환경변수 노출. **gunicorn + `app.config["DEBUG"] = False`**.

### 함정 7 — `app.secret_key` (Flask session) 와 `JWT_SECRET_KEY` 혼동
별도 키. JWT 만 쓰면 session 안 쓰지만 Flask-Login 같이 쓰면 두 키 분리.

### 함정 8 — Python 한정: `@wraps(fn)` 누락
Flask routing 이 decorator wrapper 의 `__name__` 사용 → 같은 이름 두 endpoint 등록 시 conflict. **`@wraps` 필수**.

### 함정 9 — Python 한정: decorator 안 `jwt_required()` 의 순서
`@jwt_required()` 가 가장 안쪽 (실제 view 에 가까이) 이어야 함. 바깥 decorator (`@role_required`) 가 `@jwt_required()` 후 claims 사용.

### 함정 10 — Python 한정: `os.environ["JWT_SECRET_KEY"]` KeyError
환경변수 미설정 시 import 시점에 crash → docker 가 무한 restart. **`os.environ.get(KEY)` + None 체크** 또는 startup health check.

### 함정 11 — Python 한정: `gunicorn` worker 수 와 DB pool 수
multi-worker 면 각 worker 가 own DB pool. PG `max_connections` 와 매칭 필수 — workers × pool_size ≤ max_connections.

### 함정 12 — Python 한정: `request.environ` 의 EXCEPTION_ATTR 가 다른 thread 에서 보임?
Flask 는 thread per request (default) — request context 안에서 안전. 단 `flask.g` 와 혼동 X.

---

## 12. 관련

- [[response-envelope]] — 401/403 envelope
- [[../../spring/java-spring/common/security-config|↗ Java 원본]]
- [[../../spring/kotlin-spring/common/security-config|↗ Kotlin 변형]]
- [[../api-design/api-design|↗ API 레시피]]
