---
title: "SecurityConfig + JWT + CORS — 표준 세팅 (FastAPI)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T16:40:00+09:00
tags:
  - backend
  - python-fastapi
  - common
  - security
  - jwt
  - cors
  - production
---

# SecurityConfig + JWT + CORS — 표준 세팅 (FastAPI)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + OAuth2PasswordBearer + Depends + CORSMiddleware |

**[[../python-fastapi|↑ FastAPI]]** · **[[../../spring/java-spring/common/security-config|↗ Java 원본]]** · **[[../python-flask/common/security-config|↗ Flask 변형]]**

> FastAPI 진영의 JWT + CORS + 권한 표준. 401/403 응답은 [[response-envelope]] 의 `CommonResponse` 통과. **Depends 기반 의존성 주입** 이 핵심.

---

## 1. 컴포넌트 한눈에

```
ASGI request
   │
   ├─ CORSMiddleware (응답 헤더)
   ├─ ApiLogMiddleware (응답 후 access log)
   │
   ├─ endpoint(token: str = Depends(oauth2_scheme))
   │     └─ verify_token(token) → User
   │         └─ 401: HTTPException → exception_handler → envelope
   ├─ endpoint(user: User = Depends(get_current_user))
   ├─ endpoint(user: User = Depends(require_role("ADMIN")))
   └─ 도메인 / DB

빈 (DI):
  oauth2_scheme = OAuth2PasswordBearer(...)
  Depends(get_db) — async DB session
  PasswordHasher = argon2
```

---

## 2. 표준 의존성

```bash
pip install fastapi>=0.110 \
    uvicorn[standard]>=0.27 \
    python-jose[cryptography]>=3.3 \
    passlib[argon2]>=1.7 \
    python-multipart>=0.0.9 \
    pydantic>=2.6 \
    python-dotenv
```

```toml
# pyproject.toml
[project]
dependencies = [
    "fastapi>=0.110",
    "uvicorn[standard]>=0.27",
    "python-jose[cryptography]>=3.3",
    "passlib[argon2]>=1.7",
    "python-multipart>=0.0.9",
    "pydantic>=2.6",
]
```

---

## 3. JWT util

```python
# app/common/jwt.py
from datetime import datetime, timedelta, timezone
from typing import Optional
import uuid
from jose import jwt, JWTError, ExpiredSignatureError

from .config import settings

ALGORITHM = "HS256"


def create_access_token(
    subject: str,
    user_id: int,
    role: str,
    session_id: Optional[str] = None,
) -> str:
    now = datetime.now(timezone.utc)
    expires = now + settings.JWT_ACCESS_TOKEN_EXPIRES
    payload = {
        "iss": settings.JWT_ISSUER,
        "sub": subject,                                    # email
        "iat": now,
        "exp": expires,
        "jti": str(uuid.uuid4()),
        "token_type": "ACCESS",
        "id": user_id,
        "role": role,
        "sid": session_id or str(uuid.uuid4()),
    }
    return jwt.encode(payload, settings.JWT_SECRET_KEY, algorithm=ALGORITHM)


def create_refresh_token(subject: str) -> str:
    now = datetime.now(timezone.utc)
    expires = now + settings.JWT_REFRESH_TOKEN_EXPIRES
    payload = {
        "iss": settings.JWT_ISSUER,
        "sub": subject,
        "iat": now,
        "exp": expires,
        "jti": str(uuid.uuid4()),
        "token_type": "REFRESH",
    }
    return jwt.encode(payload, settings.JWT_SECRET_KEY, algorithm=ALGORITHM)


def decode_token(token: str) -> dict:
    """jose 가 expired / invalid 자동 검증."""
    return jwt.decode(
        token,
        settings.JWT_SECRET_KEY,
        algorithms=[ALGORITHM],
        issuer=settings.JWT_ISSUER,                       # issuer 검증
        options={"leeway": 30},                           # clock skew 30s
    )
```

> **함정**: `python-jose` 의 `jwt.decode` 가 `ExpiredSignatureError` / `JWTError` 던짐. 위는 그대로 propagate 후 dependency 에서 처리.

---

## 4. Depends 기반 user 추출

```python
# app/common/deps.py
from typing import Annotated, Optional
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, ExpiredSignatureError
from sqlalchemy.ext.asyncio import AsyncSession

from .jwt import decode_token
from .exceptions import BusinessException
from .enums import ResponseCode
from .db import get_db
from app.domain.user import User
from app.repositories.user_repo import UserRepository

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/login", auto_error=False)


async def get_current_user(
    token: Annotated[Optional[str], Depends(oauth2_scheme)],
    db: Annotated[AsyncSession, Depends(get_db)],
) -> User:
    if not token:
        raise BusinessException(ResponseCode.UNAUTHORIZED, "인증이 필요합니다.")

    try:
        claims = decode_token(token)
    except ExpiredSignatureError:
        raise BusinessException(ResponseCode.INVALID_TOKEN, "만료된 토큰입니다.")
    except JWTError as e:
        raise BusinessException(ResponseCode.INVALID_TOKEN, f"유효하지 않은 토큰: {e}")

    if claims.get("token_type") != "ACCESS":
        raise BusinessException(ResponseCode.INVALID_TOKEN, "ACCESS 토큰이 아닙니다.")

    email = claims.get("sub")
    if not email:
        raise BusinessException(ResponseCode.INVALID_TOKEN, "토큰 subject 없음")

    repo = UserRepository(db)
    user = await repo.find_active_by_email(email)
    if user is None:
        raise BusinessException(ResponseCode.USER_NOT_FOUND)

    # 세션 무효화 검증 (선택)
    sid = claims.get("sid")
    if sid and not await repo.is_session_active(user.id, sid):
        raise BusinessException(ResponseCode.INVALID_TOKEN, "세션이 무효화되었습니다.")

    return user


async def get_optional_user(
    token: Annotated[Optional[str], Depends(oauth2_scheme)],
    db: Annotated[AsyncSession, Depends(get_db)],
) -> Optional[User]:
    """토큰 있으면 user, 없으면 None — 익명 허용 endpoint 용."""
    if not token:
        return None
    try:
        return await get_current_user(token, db)
    except BusinessException:
        return None
```

> **FastAPI idiom 포인트**:
> - `Annotated[X, Depends(...)]` — pydantic v2 / FastAPI 0.95+ 권장 형식 (PEP 593).
> - `OAuth2PasswordBearer` — Swagger UI 에 `Authorize` 버튼 자동 생성.
> - `auto_error=False` — 토큰 없을 때 자동 401 던지지 않고 None 반환 (커스텀 처리).

---

## 5. 권한 — `require_role` factory

```python
# app/common/deps.py 에 추가
from typing import Iterable


def require_role(*allowed_roles: str):
    """role 기반 권한 — Spring @PreAuthorize("hasAnyRole(...)") 동등."""
    async def _checker(user: Annotated[User, Depends(get_current_user)]) -> User:
        if user.role.name not in allowed_roles:
            raise BusinessException(
                ResponseCode.INSUFFICIENT_PERMISSION,
                "해당 리소스에 접근할 권한이 없습니다.",
            )
        return user
    return _checker


# 사용
from typing import Annotated
from fastapi import APIRouter, Depends

router = APIRouter(prefix="/admin", tags=["admin"])


@router.post("/products/{product_id}/hide")
async def admin_hide_product(
    product_id: str,
    user: Annotated[User, Depends(require_role("ADMIN", "MASTER"))],
):
    await product_service.hide(product_id)
    return CommonResponse.success(ResponseCode.OK, "숨김 처리")
```

> **idiom**: factory 함수가 dependency 를 반환 — Spring 의 SpEL `@PreAuthorize("hasRole(...)")` 보다 type-safe.

---

## 6. ownership 검증

```python
async def owns_or_admin_product(
    product_id: str,
    user: Annotated[User, Depends(get_current_user)],
    db: Annotated[AsyncSession, Depends(get_db)],
) -> User:
    if user.role.name in ("ADMIN", "MASTER"):
        return user
    product = await ProductRepository(db).find_by_id(product_id)
    if product is None or product.seller_id != user.id:
        raise BusinessException(ResponseCode.FORBIDDEN, "본인의 상품만 접근 가능합니다.")
    return user


@router.patch("/products/{product_id}")
async def update_product(
    product_id: str,
    payload: UpdateProductRequest,
    user: Annotated[User, Depends(owns_or_admin_product)],
):
    await product_service.update(product_id, payload)
    return ok("수정 완료")
```

---

## 7. CORSMiddleware

```python
# app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from .config import settings

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.CORS_ORIGINS,                 # 명시 (와일드카드 X)
    allow_credentials=False,                              # JWT + Authorization 헤더면 False
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
    allow_headers=["Authorization", "Content-Type", "Accept-Language", "X-DEVICE-UUID"],
    max_age=3600,
)
```

---

## 8. Config (pydantic Settings)

```python
# app/common/config.py
from datetime import timedelta
from pydantic import BaseModel
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8", extra="ignore")

    # JWT
    JWT_SECRET_KEY: str
    JWT_ACCESS_TOKEN_EXPIRES: timedelta = timedelta(minutes=15)
    JWT_REFRESH_TOKEN_EXPIRES: timedelta = timedelta(days=14)
    JWT_ISSUER: str = "shop.example.com"

    # CORS
    CORS_ORIGINS: list[str] = ["http://localhost:3000"]

    # DB
    DB_URL: str


settings = Settings()
```

> **pydantic-settings v2 idiom**: `BaseSettings` 가 env / `.env` 자동 로드 + type 변환 + 검증.

---

## 9. 로그인 endpoint

```python
# app/api/auth.py
from typing import Annotated
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession

from app.common.deps import get_db
from app.common.dto import CommonResponse
from app.common.enums import ResponseCode
from app.common.exceptions import BusinessException
from app.common.jwt import create_access_token, create_refresh_token
from app.common.password import verify_password
from app.repositories.user_repo import UserRepository

router = APIRouter(prefix="/api/v1/auth", tags=["auth"])


class LoginRequest(BaseModel):
    email: EmailStr
    password: str


class TokenResponse(BaseModel):
    accessToken: str
    refreshToken: str
    tokenType: str = "Bearer"
    expiresIn: int = 900


@router.post("/login", response_model=CommonResponse[TokenResponse])
async def login(
    payload: LoginRequest,
    db: Annotated[AsyncSession, Depends(get_db)],
) -> CommonResponse[TokenResponse]:
    user = await UserRepository(db).find_active_by_email(payload.email)
    if user is None or not verify_password(payload.password, user.password_hash):
        raise BusinessException(
            ResponseCode.UNAUTHORIZED,
            "입력하신 이메일과 비밀번호를 확인해 주세요.",
        )

    access = create_access_token(
        subject=user.email,
        user_id=user.id,
        role=user.role.name,
    )
    refresh = create_refresh_token(subject=user.email)

    return CommonResponse.success(
        ResponseCode.OK,
        "로그인 성공",
        TokenResponse(accessToken=access, refreshToken=refresh),
    )
```

---

## 10. 비밀번호 해시 — Argon2 (passlib)

```python
# app/common/password.py
from passlib.context import CryptContext

_pwd_context = CryptContext(
    schemes=["argon2", "bcrypt"],
    deprecated="auto",
    argon2__time_cost=3,
    argon2__memory_cost=65536,        # 64 MB
    argon2__parallelism=2,
)


def hash_password(plain: str) -> str:
    return _pwd_context.hash(plain)


def verify_password(plain: str, hashed: str) -> bool:
    return _pwd_context.verify(plain, hashed)


def needs_rehash(hashed: str) -> bool:
    return _pwd_context.needs_update(hashed)
```

---

## 11. 운영 체크리스트

- [ ] `JWT_SECRET_KEY` Base64 256-bit+ — `.env` 또는 vault
- [ ] `JWT_ACCESS_TOKEN_EXPIRES` 15분
- [ ] CORS `allow_origins` 명시 (와일드카드 X)
- [ ] `allow_credentials=False` + Authorization 헤더
- [ ] `get_current_user` 가 BusinessException 던지면 envelope 응답
- [ ] 모든 protected endpoint 에 `Depends(get_current_user)` 또는 `Depends(require_role(...))`
- [ ] `response_model=CommonResponse[X]` 명시 — OpenAPI schema
- [ ] uvicorn `--workers` × DB pool ≤ DB `max_connections`
- [ ] passlib Argon2 cost (time_cost=3, memory_cost=65536)
- [ ] `pydantic-settings` 로 env 검증 (운영 boot 시 missing key fail-fast)

---

## 12. 함정 모음 (FastAPI + Python 한정)

### 함정 1 — `OAuth2PasswordBearer` 의 `auto_error=True` (default)
토큰 없을 때 FastAPI 가 자동 401 (raw `{"detail":"Not authenticated"}`) → envelope 깨짐. **`auto_error=False`** + custom dependency 로 BusinessException 던지기.

### 함정 2 — `python-jose` 의 `jwt.decode` clock skew
default 0초 — 1초 차이로 fail. `options={"leeway": 30}` 명시.

### 함정 3 — `Depends(get_db)` 가 generator 가 아님
async session 은 `async with` 또는 `yield` 패턴 필요. SQLAlchemy 2.0+ 의 `AsyncSession` 정확히.

### 함정 4 — `BaseSettings` 가 환경변수 미설정 시 import-time crash
`JWT_SECRET_KEY: str` (default 없음) 가 환경변수 없으면 `ValidationError`. docker compose `env_file` 설정 확인.

### 함정 5 — pydantic v2 의 `EmailStr` 의존성
`from pydantic import EmailStr` 시 `email-validator` 패키지 필수 — 별도 `pip install email-validator`.

### 함정 6 — async middleware 안 sync 호출
이벤트 loop block — `await` 누락 시 deadlock 가능. async 함수만.

### 함정 7 — Python 한정: `Annotated[X, Depends(...)]` vs `X = Depends(...)`
Pydantic v2 / FastAPI 0.95+ 는 `Annotated` 권장 — IDE / mypy 가 더 정확히 추론. 단 옛 코드는 `X = Depends(...)` 도 OK.

### 함정 8 — Python 한정: `Depends(require_role("ADMIN"))` 가 매 요청마다 새 factory call
`require_role` 자체는 cheap (closure) 이지만 안의 `get_current_user` 가 DB 호출. **rate-limit + cache** 검토.

### 함정 9 — Python 한정: `passlib` 의 `bcrypt` 4.x 호환 이슈
`passlib 1.7.4` + `bcrypt 4.x` 가 WARNING 출력. `bcrypt==3.2.2` lock 또는 `argon2` 전용.

### 함정 10 — Python 한정: uvicorn `--reload` 가 운영
dev only. 운영은 `gunicorn -k uvicorn.workers.UvicornWorker` + `--workers` 명시.

### 함정 11 — Python 한정: ASGI app 객체가 module-level
multi-worker 가 동시 import — module-level side effect (DB connection 등) 가 worker 마다. **lifespan event** (startup/shutdown) 으로 격리.

---

## 13. 관련

- [[response-envelope]] — 401/403 envelope
- [[../../spring/java-spring/common/security-config|↗ Java 원본]]
- [[../python-flask/common/security-config|↗ Flask 변형]]
- [[../api-design/api-design|↗ API 레시피]]
