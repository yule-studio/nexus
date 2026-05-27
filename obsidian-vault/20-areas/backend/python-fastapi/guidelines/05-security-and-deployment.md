---
title: "05. JWT + Argon2 + OAuth2PasswordBearer + Docker"
kind: guide
status: current
created_at: 2026-05-27T15:30:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, guidelines, security, jwt, deployment, docker]
---

# 05. JWT + Argon2 + OAuth2PasswordBearer + Docker

**[[guidelines|↑ Hub]]** ← [[04-controller-and-validation]] · → 다음 [[06-swagger-openapi]]

> python-jose JWT + passlib Argon2 + OAuth2PasswordBearer Depends + Docker.

---

## 1. JWT util

```python
# app/services/jwt_util.py
from datetime import datetime, timezone
from typing import Optional
import uuid

from jose import jwt, JWTError, ExpiredSignatureError

from app.config import settings


def create_access_token(subject: str, additional: Optional[dict] = None) -> str:
    now = datetime.now(timezone.utc)
    payload = {
        "iss": settings.JWT_ISSUER,
        "sub": subject,
        "iat": now,
        "exp": now + settings.JWT_ACCESS_TOKEN_EXPIRES,
        "jti": str(uuid.uuid4()),
        "token_type": "ACCESS",
        **(additional or {}),
    }
    return jwt.encode(payload, settings.JWT_SECRET_KEY, algorithm=settings.JWT_ALGORITHM)


def create_refresh_token(subject: str) -> str:
    now = datetime.now(timezone.utc)
    payload = {
        "iss": settings.JWT_ISSUER,
        "sub": subject,
        "iat": now,
        "exp": now + settings.JWT_REFRESH_TOKEN_EXPIRES,
        "jti": str(uuid.uuid4()),
        "token_type": "REFRESH",
    }
    return jwt.encode(payload, settings.JWT_SECRET_KEY, algorithm=settings.JWT_ALGORITHM)


def decode_token(token: str) -> dict:
    return jwt.decode(
        token,
        settings.JWT_SECRET_KEY,
        algorithms=[settings.JWT_ALGORITHM],
        issuer=settings.JWT_ISSUER,
        options={"leeway": 30},                     # ⚠️ clock skew 30s
    )
```

---

## 2. 비밀번호 해시 — passlib + Argon2

```python
# app/services/password.py
from passlib.context import CryptContext

_pwd = CryptContext(schemes=["argon2"], deprecated="auto")


def hash_password(plain: str) -> str:
    return _pwd.hash(plain)


def verify_password(plain: str, hashed: str) -> bool:
    return _pwd.verify(plain, hashed)


def needs_rehash(hashed: str) -> bool:
    return _pwd.needs_update(hashed)
```

---

## 3. Depends — get_current_user

```python
# app/deps.py
from typing import Annotated, AsyncGenerator
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, ExpiredSignatureError
from sqlalchemy.ext.asyncio import AsyncSession

from app.db import get_db
from app.domain.user import User
from app.services.jwt_util import decode_token
from app.errors import UnauthorizedException, NotFoundException


# tokenUrl = login endpoint (Swagger UI 의 Authorize 에 표시)
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/login", auto_error=False)


SessionDep = Annotated[AsyncSession, Depends(get_db)]
TokenDep = Annotated[str | None, Depends(oauth2_scheme)]


async def get_current_user(token: TokenDep, db: SessionDep) -> User:
    if not token:
        raise UnauthorizedException("인증이 필요합니다.")
    try:
        claims = decode_token(token)
    except ExpiredSignatureError:
        raise UnauthorizedException("만료된 토큰입니다.")
    except JWTError as e:
        raise UnauthorizedException(f"유효하지 않은 토큰: {e}")

    if claims.get("token_type") != "ACCESS":
        raise UnauthorizedException("ACCESS 토큰이 아닙니다.")

    user_id = int(claims["sub"])
    user = await db.get(User, user_id)
    if user is None:
        raise NotFoundException("사용자")
    return user


CurrentUserDep = Annotated[User, Depends(get_current_user)]


async def get_optional_user(token: TokenDep, db: SessionDep) -> User | None:
    """익명 허용 endpoint 용."""
    if not token:
        return None
    try:
        return await get_current_user(token, db)
    except UnauthorizedException:
        return None
```

> **`OAuth2PasswordBearer(auto_error=False)`** = 토큰 없을 때 자동 401 안 함. custom dependency 가 `UnauthorizedException` 던지도록 envelope 통일.

---

## 4. require_role — 권한 factory

```python
# app/deps.py 에 추가
def require_role(*allowed: str):
    """role 기반 권한 — Spring @PreAuthorize 동등."""

    async def _checker(user: CurrentUserDep) -> User:
        if user.role not in allowed:
            from app.errors import BusinessException
            raise BusinessException(403, "INSUFFICIENT_PERMISSION",
                                    "해당 리소스에 접근할 권한이 없습니다.")
        return user
    return _checker
```

```python
@router.delete("/admin/users/{id}")
async def admin_delete_user(
    id: int,
    admin: Annotated[User, Depends(require_role("ADMIN", "MASTER"))],
    db: SessionDep,
):
    # ...
    return {"deleted": id}
```

---

## 5. Auth router — signup / login

```python
# app/api/auth.py
from typing import Annotated
from fastapi import APIRouter, status
from sqlalchemy import select

from app.deps import SessionDep
from app.schemas.auth import SignupRequest, LoginRequest, TokenResponse, UserResponse
from app.schemas.envelope import ApiResponse
from app.domain.user import User
from app.services.password import hash_password, verify_password
from app.services.jwt_util import create_access_token, create_refresh_token
from app.errors import ConflictException, UnauthorizedException

router = APIRouter()


@router.post(
    "/signup",
    response_model=ApiResponse[UserResponse],
    status_code=status.HTTP_201_CREATED,
)
async def signup(payload: SignupRequest, db: SessionDep):
    existing = await db.scalar(select(User).where(User.email == payload.email))
    if existing is not None:
        raise ConflictException("이미 가입된 이메일입니다.")

    user = User(
        email=payload.email,
        password_hash=hash_password(payload.password),
        name=payload.name,
    )
    db.add(user)
    await db.commit()
    await db.refresh(user)
    return ApiResponse.ok(UserResponse.model_validate(user), "회원가입 성공")


@router.post("/login", response_model=ApiResponse[TokenResponse])
async def login(payload: LoginRequest, db: SessionDep):
    user = await db.scalar(select(User).where(User.email == payload.email))
    if user is None or not verify_password(payload.password, user.password_hash):
        raise UnauthorizedException("이메일 또는 비밀번호가 올바르지 않습니다.")

    return ApiResponse.ok(
        TokenResponse(
            accessToken=create_access_token(
                subject=str(user.id),
                additional={"email": user.email, "role": "USER"},
            ),
            refreshToken=create_refresh_token(subject=str(user.id)),
            tokenType="Bearer",
            expiresIn=900,
        ),
        "로그인 성공",
    )
```

```python
# app/schemas/auth.py
from pydantic import BaseModel, EmailStr, Field, ConfigDict


class SignupRequest(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8, max_length=64)
    name: str = Field(min_length=1, max_length=100)


class LoginRequest(BaseModel):
    email: EmailStr
    password: str


class TokenResponse(BaseModel):
    accessToken: str
    refreshToken: str
    tokenType: str = "Bearer"
    expiresIn: int = 900


class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    email: EmailStr
    name: str
```

---

## 6. 실습

```bash
fastapi dev app/main.py

# 1. signup
curl -X POST http://localhost:8000/api/v1/auth/signup \
  -H "Content-Type: application/json" \
  -d '{"email":"yuchan@example.com","password":"P@ssw0rd1!","name":"yuchan"}'

# 2. login → token
TOKEN=$(curl -sX POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"yuchan@example.com","password":"P@ssw0rd1!"}' \
  | jq -r '.data.accessToken')

# 3. 메모 생성
curl -X POST http://localhost:8000/api/v1/memos \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"첫 메모","content":"안녕"}'

# 4. Swagger UI 에서도 (Authorize 버튼)
open http://localhost:8000/docs
```

---

## 7. Dockerfile

```dockerfile
FROM python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
        libpq5 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app/ ./app/
COPY alembic.ini .
COPY migrations/ ./migrations/

RUN useradd -m -u 1000 app && chown -R app:app /app
USER app

ENV PORT=8000 PYTHONUNBUFFERED=1
EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

### 7.1 docker-compose

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
      JWT_SECRET_KEY: dev-jwt-32-bytes-please-please-change
      DATABASE_URL: postgresql+asyncpg://memo:memo@db:5432/memo
    ports: ["8000:8000"]
    depends_on: [db]
    command: >
      sh -c "alembic upgrade head &&
             uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4"

volumes:
  dbdata:
```

---

## 8. 운영 체크리스트

- [ ] `SECRET_KEY` / `JWT_SECRET_KEY` vault / Secrets Manager
- [ ] HTTPS (Nginx / ALB)
- [ ] `uvicorn --workers N` 인스턴스 spec
- [ ] PG `max_connections` ≥ workers × pool_size
- [ ] `alembic upgrade head` container startup
- [ ] log → stdout
- [ ] healthz endpoint
- [ ] CORS `allow_origins` 명시
- [ ] passlib Argon2 (cost 적정)
- [ ] JWT secret 32 bytes+
- [ ] `auto_error=False` + custom dependency (envelope 통일)

---

## 9. 자주 막히는 함정

### 함정 1 — `OAuth2PasswordBearer(auto_error=True)` (default)
토큰 없을 때 FastAPI 자동 raw `{"detail":"Not authenticated"}` 응답 → envelope 깨짐. **`auto_error=False`**.

### 함정 2 — `python-jose` clock skew
default 0 — 1초 차이로 fail. `options={"leeway": 30}`.

### 함정 3 — `passlib bcrypt 4.x` 호환 경고
`passlib 1.7.4` + `bcrypt 4.x` warning. argon2 권장 (본 가이드).

### 함정 4 — `uvicorn --reload` 운영
dev only. 운영 `--workers N`.

### 함정 5 — JWT secret 짧음
HMAC-SHA256 약함. 32 bytes+.

### 함정 6 — `Depends(require_role(...))` 가 매 호출마다 closure
`require_role` 자체는 cheap (closure). 내부 `get_current_user` 가 DB 호출 — 정상.

### 함정 7 — `Settings()` import-time crash
`.env` 없으면 fail. ([[02-project-structure#9 함정 1|02 §9 함정 1]])

### 함정 8 — `gunicorn -k uvicorn.workers.UvicornWorker` vs `uvicorn --workers`
둘 다 OK. gunicorn 이 process 관리 (signal / graceful) 약간 우위. 본 가이드는 uvicorn 직접.

### 함정 9 — multi-worker = 별도 cache
process 분리 — `lru_cache` 가 worker 별. **Redis 같은 외부 저장소**.

### 함정 10 — `from_attributes=True` 누락
ORM 객체 → pydantic 변환 시 ValidationError. ConfigDict 에 추가.

---

## 10. 다음

✅ Security + 배포 끝.

→ [[06-swagger-openapi]] — 자동 /docs + customize + tags / Authorize
