---
title: "FastAPI Getting Started — 0 부터 첫 프로젝트 배포까지"
kind: guide
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T20:00:00+09:00
tags:
  - backend
  - python-fastapi
  - getting-started
  - guide
  - tutorial
---

# FastAPI Getting Started — 0 부터 첫 프로젝트 배포까지

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | 설치 → 첫 프로젝트 → CRUD API → 배포 단일 가이드 |

**[[python-fastapi|↑ FastAPI]]** · **[[../python-flask/getting-started|↗ Flask 가이드]]**

> **이 가이드를 보고 처음부터 FastAPI 프로젝트를 만들고 배포까지 할 수 있도록 설계.**
> [[../python-flask/getting-started|Flask 가이드]] 와 같은 가상 프로젝트 (메모 API) 를 FastAPI 로 — 두 가이드 나란히 비교하면 차이 명확.
> 사전 지식: Python 기본 + HTTP/JSON/DB 약간. Flask 안 봐도 OK.

---

## 0. FastAPI 가 무엇인가

**FastAPI** = Python 의 **modern async 웹 프레임워크** (2018+). Flask 의 단순함 + Spring 의 type safety + Swagger 자동 생성.

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()


class Greeting(BaseModel):
    message: str


@app.get("/hello/{name}", response_model=Greeting)
async def hello(name: str) -> Greeting:
    return Greeting(message=f"Hello, {name}!")

# uvicorn main:app 으로 실행 → http://localhost:8000/hello/yuchan
# 자동으로 http://localhost:8000/docs (Swagger UI) 생성!
```

### FastAPI 의 4가지 핵심 특징

1. **Type hints 기반** — Python 의 type hint (`name: str`) 가 그대로 validation / OpenAPI schema 가 됨.
2. **async / await** — `async def` endpoint = non-blocking. 동시 수천 요청 처리.
3. **Pydantic** — request/response 가 자동으로 검증 + 직렬화.
4. **자동 OpenAPI / Swagger UI** — `/docs` 에 항상 최신 스펙. Postman 없이 바로 시험.

### Flask vs FastAPI — 한 줄

| | Flask | FastAPI |
| --- | --- | --- |
| 컨셉 | sync micro framework | async modern framework |
| Type hint | 선택 (작동 무관) | **필수** (validation / docs 핵심) |
| 입력 검증 | marshmallow / pydantic 별도 | **pydantic 자동** |
| 의존성 주입 | manual | **`Depends`** built-in |
| Swagger | 별도 설치 (flask-smorest 등) | **자동 `/docs`** |
| WSGI vs ASGI | WSGI (gunicorn) | ASGI (uvicorn / hypercorn) |
| 비동기 DB | 불편 | 자연스러움 (SQLAlchemy 2.x async) |
| Spring 비유 | Spring MVC | Spring WebFlux + Spring Boot autoconfig |

### 0.1 무엇을 만들 건가

Flask 가이드와 같은 **메모 API**:

```
GET    /api/v1/memos               # 목록
POST   /api/v1/memos               # 생성
GET    /api/v1/memos/{id}          # 단건
PATCH  /api/v1/memos/{id}          # 수정
DELETE /api/v1/memos/{id}          # 삭제

POST   /api/v1/auth/signup
POST   /api/v1/auth/login
```

다만 **async + Depends + pydantic** 으로 다시.

---

## 1. 사전 준비

### 1.1 Python 3.12 + venv

```bash
mkdir fastapi-memo-api && cd fastapi-memo-api
python3.12 -m venv .venv
source .venv/bin/activate            # macOS / Linux
# .venv\Scripts\activate              # Windows
```

(자세한 설치는 [[../python-flask/getting-started#1 사전 준비|Flask §1]] 와 동일)

### 1.2 Editor

VS Code + Python 확장. **type hint 가 핵심이라 IDE 의 type checker (Pylance) 가 강력하게 도와줌**.

---

## 2. Hello FastAPI

### 2.1 설치

```bash
pip install --upgrade pip
pip install "fastapi[standard]"      # standard 가 uvicorn / 등 포함
# 또는 분리:
# pip install fastapi uvicorn[standard]

pip freeze > requirements.txt
```

`fastapi[standard]` 설치 내용:
```
fastapi==0.110.x
fastapi-cli (개발 편의)
uvicorn[standard]   (ASGI server)
python-multipart    (form upload)
email-validator     (EmailStr)
jinja2              (template, 선택)
```

### 2.2 main.py

```python
# main.py
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI(
    title="Memo API",
    description="첫 FastAPI 프로젝트",
    version="1.0.0",
)


class Greeting(BaseModel):
    message: str


@app.get("/", response_model=Greeting)
async def index() -> Greeting:
    return Greeting(message="Hello, FastAPI!")


@app.get("/hello/{name}", response_model=Greeting)
async def hello(name: str) -> Greeting:
    return Greeting(message=f"Hello, {name}!")
```

### 2.3 실행

```bash
# fastapi CLI (0.110+ 신규)
fastapi dev main.py

# 또는 uvicorn 직접
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

출력:
```
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
INFO:     Application startup complete.
```

테스트:
```bash
curl http://localhost:8000/
# {"message":"Hello, FastAPI!"}

curl http://localhost:8000/hello/yuchan
# {"message":"Hello, yuchan!"}
```

### 2.4 Swagger UI / ReDoc 자동 생성

브라우저:
- **<http://localhost:8000/docs>** — Swagger UI (요청/응답 시험 가능)
- **<http://localhost:8000/redoc>** — ReDoc (읽기 친화)
- **<http://localhost:8000/openapi.json>** — OpenAPI spec JSON

> **이게 FastAPI 의 킬러 기능** — 코드 변경하면 즉시 docs 도 변경. FE 가 항상 최신 API 보고 작업.

### 2.5 핵심 개념 — Path Parameter / Query / Type Conversion

```python
from fastapi import FastAPI
from typing import Optional

app = FastAPI()


@app.get("/items/{item_id}")
async def get_item(item_id: int, q: Optional[str] = None, limit: int = 20):
    return {
        "item_id": item_id,            # path → int 자동 변환
        "q": q,                         # ?q=... → Optional[str]
        "limit": limit,                 # ?limit=10 → int, 없으면 20
    }


@app.post("/items")
async def create_item(item: dict):
    return {"received": item}
```

```bash
curl "http://localhost:8000/items/42?q=foo&limit=5"
# {"item_id":42,"q":"foo","limit":5}

curl "http://localhost:8000/items/abc"   # ⚠️ int 변환 실패 → 422 자동
# {"detail":[{"type":"int_parsing","loc":["path","item_id"],"msg":"..."}]}
```

> **type hint 가 곧 validation**. `item_id: int` 면 FastAPI 가 자동으로 422.

---

## 3. 프로젝트 구조 + Settings

### 3.1 디렉터리

```
fastapi-memo-api/
├── .venv/
├── .env
├── .gitignore
├── requirements.txt
├── pyproject.toml
├── README.md
├── app/
│   ├── __init__.py
│   ├── main.py                  # FastAPI() 인스턴스 + 라우터 include
│   ├── config.py                # Settings (pydantic-settings)
│   ├── deps.py                  # Depends 함수들 (DB session / get_current_user / ...)
│   ├── db.py                    # SQLAlchemy engine + session
│   ├── domain/                  # ORM 모델
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── memo.py
│   ├── schemas/                 # pydantic request/response
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   └── memo.py
│   ├── api/                     # APIRouter (Flask blueprint 동등)
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   └── memos.py
│   ├── services/                # business logic
│   │   ├── __init__.py
│   │   ├── auth_service.py
│   │   ├── memo_service.py
│   │   └── password.py
│   ├── errors.py                # custom Exception + handler
│   └── middlewares.py           # CORS / 로깅
├── alembic.ini
├── migrations/                  # Alembic
└── tests/
    ├── conftest.py
    └── test_memos.py
```

### 3.2 requirements.txt

```txt
fastapi[standard]==0.110.0
SQLAlchemy[asyncio]==2.0.29
asyncpg==0.29.0              # async PG driver
alembic==1.13.1
python-jose[cryptography]==3.3.0
passlib[argon2]==1.7.4
argon2-cffi==23.1.0
pydantic==2.6.4
pydantic-settings==2.2.1
python-dotenv==1.0.1
httpx==0.27.0                # 테스트용
pytest==8.1.1
pytest-asyncio==0.23.6
```

```bash
pip install -r requirements.txt
```

### 3.3 `.env` + Settings (pydantic-settings)

```bash
# .env
SECRET_KEY=dev-secret-change-in-prod
JWT_SECRET_KEY=dev-jwt-secret-change-in-prod
DATABASE_URL=postgresql+asyncpg://memo:memo@localhost:5432/memo
CORS_ORIGINS=http://localhost:3000
```

```python
# app/config.py
from datetime import timedelta
from typing import List
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    SECRET_KEY: str
    JWT_SECRET_KEY: str
    JWT_ACCESS_TOKEN_EXPIRES: timedelta = timedelta(minutes=15)
    JWT_REFRESH_TOKEN_EXPIRES: timedelta = timedelta(days=14)
    JWT_ALGORITHM: str = "HS256"
    JWT_ISSUER: str = "memo-api"

    DATABASE_URL: str
    CORS_ORIGINS: List[str] = ["http://localhost:3000"]


settings = Settings()
```

> **pydantic-settings**: `.env` / 시스템 env 자동 로드 + type 변환 + 검증. Settings 미설정 (`DATABASE_URL`) 시 import 시점에 `ValidationError`.

### 3.4 main.py — app factory

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from .config import settings
from .errors import register_error_handlers
from .db import engine
from .api import auth, memos


@asynccontextmanager
async def lifespan(app: FastAPI):
    """startup / shutdown 이벤트."""
    # startup
    print("FastAPI starting...")
    yield
    # shutdown
    await engine.dispose()
    print("FastAPI stopped.")


def create_app() -> FastAPI:
    app = FastAPI(
        title="Memo API",
        version="1.0.0",
        lifespan=lifespan,
    )

    # CORS
    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.CORS_ORIGINS,
        allow_credentials=False,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    # error handlers
    register_error_handlers(app)

    # routers
    app.include_router(auth.router, prefix="/api/v1/auth", tags=["auth"])
    app.include_router(memos.router, prefix="/api/v1/memos", tags=["memos"])

    return app


app = create_app()
```

> **`lifespan`** — Spring 의 `ApplicationListener<ContextRefreshedEvent>` 동등. DB pool / Redis 연결 정리.
> **`include_router(prefix=..., tags=...)`** — Flask Blueprint 동등 + Swagger 태그.

---

## 4. DB — SQLAlchemy 2.x async

### 4.1 PostgreSQL 준비

(Flask 가이드 [[../python-flask/getting-started#4 Domain 데이터 model|§4.1]] 와 동일 — docker / brew)

### 4.2 engine + session

```python
# app/db.py
from typing import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase

from .config import settings

engine = create_async_engine(
    settings.DATABASE_URL,
    echo=False,                  # True 면 모든 SQL log
    pool_pre_ping=True,
    pool_size=20,
    max_overflow=10,
)

AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
)


class Base(DeclarativeBase):
    pass


async def get_db() -> AsyncGenerator[AsyncSession, None]:
    """FastAPI Depends 로 주입되는 session."""
    async with AsyncSessionLocal() as session:
        yield session
```

> **`async with AsyncSessionLocal() as session`** — request 끝나면 자동 close. Spring `@RequestScope` 동등.

### 4.3 Model

```python
# app/domain/user.py
from datetime import datetime
from sqlalchemy import String, DateTime, func
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.db import Base


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    password_hash: Mapped[str] = mapped_column(String(255))
    name: Mapped[str] = mapped_column(String(100))
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )

    memos: Mapped[list["Memo"]] = relationship(
        back_populates="owner",
        cascade="all, delete-orphan",
        lazy="selectin",                 # async 호환 — N+1 회피
    )

    def __repr__(self) -> str:
        return f"<User id={self.id} email={self.email!r}>"


# app/domain/memo.py
from datetime import datetime
from typing import TYPE_CHECKING
from sqlalchemy import String, Text, DateTime, ForeignKey, func
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.db import Base

if TYPE_CHECKING:
    from .user import User


class Memo(Base):
    __tablename__ = "memos"

    id: Mapped[int] = mapped_column(primary_key=True)
    owner_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    title: Mapped[str] = mapped_column(String(200))
    content: Mapped[str] = mapped_column(Text, default="")
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now(),
    )

    owner: Mapped["User"] = relationship(back_populates="memos", lazy="selectin")
```

> **SQLAlchemy 2.x `Mapped` / `mapped_column`** — type hint 기반 declarative. Java JPA 의 `@Entity` 와 비슷한 명확성.
> **`lazy="selectin"`** — N+1 회피. async 와 호환되는 fetch 전략.

### 4.4 Alembic 마이그레이션

```bash
alembic init -t async migrations         # async template
```

```python
# alembic.ini — sqlalchemy.url 은 dotenv 로 동적 주입 가능
# migrations/env.py 수정 — Base.metadata 연결
```

```python
# migrations/env.py 상단
import asyncio
from logging.config import fileConfig
from sqlalchemy.ext.asyncio import async_engine_from_config
from sqlalchemy import pool
from alembic import context

from app.config import settings
from app.db import Base
from app.domain import user, memo                       # noqa: F401 — 모델 인지

config = context.config
config.set_main_option("sqlalchemy.url", settings.DATABASE_URL)

target_metadata = Base.metadata


def run_migrations_offline() -> None:
    url = config.get_main_option("sqlalchemy.url")
    context.configure(url=url, target_metadata=target_metadata, literal_binds=True)
    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection) -> None:
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


async def run_migrations_online() -> None:
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()


if context.is_offline_mode():
    run_migrations_offline()
else:
    asyncio.run(run_migrations_online())
```

```bash
# 첫 마이그레이션 생성
alembic revision --autogenerate -m "create users and memos"

# 적용
alembic upgrade head
```

확인:
```bash
psql postgresql://memo:memo@localhost:5432/memo -c "\dt"
```

---

## 5. Pydantic Schema — request / response

```python
# app/schemas/auth.py
from pydantic import BaseModel, EmailStr, Field


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
    id: int
    email: EmailStr
    name: str

    model_config = {"from_attributes": True}             # ORM → pydantic 자동 변환
```

```python
# app/schemas/memo.py
from datetime import datetime
from typing import Optional
from pydantic import BaseModel, Field, ConfigDict


class CreateMemoRequest(BaseModel):
    title: str = Field(min_length=1, max_length=200)
    content: str = Field(default="", max_length=10_000)


class UpdateMemoRequest(BaseModel):
    title: Optional[str] = Field(default=None, min_length=1, max_length=200)
    content: Optional[str] = Field(default=None, max_length=10_000)


class MemoResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    title: str
    content: str
    createdAt: datetime = Field(alias="created_at")
    updatedAt: datetime = Field(alias="updated_at")


class MemoListResponse(BaseModel):
    items: list[MemoResponse]
    total: int
```

> **`from_attributes=True`** — ORM 객체 (`Memo` instance) 를 그대로 pydantic 으로 변환 가능. `MemoResponse.model_validate(memo)` OK.
> **`Field(alias="created_at")`** — DB column name 과 JSON field name 다를 때.

---

## 6. 에러 처리

```python
# app/errors.py
import logging
from typing import Any
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from starlette.exceptions import HTTPException as StarletteHTTPException

log = logging.getLogger(__name__)


class BusinessException(Exception):
    def __init__(self, status: int, code: str, message: str) -> None:
        self.status = status
        self.code = code
        self.message = message
        super().__init__(message)


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


def _body(status: int, code: str, message: str, details: Any = None) -> dict:
    body = {"success": False, "error": {"code": code, "message": message}}
    if details is not None:
        body["error"]["details"] = details
    return body


def register_error_handlers(app: FastAPI) -> None:

    @app.exception_handler(BusinessException)
    async def handle_business(request: Request, ex: BusinessException) -> JSONResponse:
        if ex.status >= 500:
            log.error("BIZ 5xx: %s", ex, exc_info=True)
        else:
            log.warning("BIZ %s: %s", ex.code, ex.message)
        return JSONResponse(
            status_code=ex.status,
            content=_body(ex.status, ex.code, ex.message),
        )

    @app.exception_handler(RequestValidationError)
    async def handle_validation(request: Request, ex: RequestValidationError) -> JSONResponse:
        log.warning("VALIDATION: %s", ex.errors())
        # field 별 에러 맵
        details = {}
        for err in ex.errors():
            field = ".".join(str(loc) for loc in err["loc"][1:]) or err["loc"][0]
            details[str(field)] = err["msg"]
        return JSONResponse(
            status_code=422,
            content=_body(422, "VALIDATION_ERROR", "입력 형식이 올바르지 않습니다.", details),
        )

    @app.exception_handler(StarletteHTTPException)
    async def handle_http(request: Request, ex: StarletteHTTPException) -> JSONResponse:
        return JSONResponse(
            status_code=ex.status_code,
            content=_body(ex.status_code, "HTTP_ERROR", str(ex.detail) if ex.detail else ""),
        )

    @app.exception_handler(Exception)
    async def handle_any(request: Request, ex: Exception) -> JSONResponse:
        log.exception("UNCAUGHT")
        return JSONResponse(
            status_code=500,
            content=_body(500, "INTERNAL_SERVER_ERROR", "서버 내부 오류가 발생했습니다."),
        )
```

---

## 7. 인증 — JWT + Depends

### 7.1 JWT util

```python
# app/services/jwt_util.py
from datetime import datetime, timedelta, timezone
import uuid
from typing import Optional
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
        options={"leeway": 30},
    )
```

### 7.2 비밀번호 해시

```python
# app/services/password.py
from passlib.context import CryptContext

_pwd = CryptContext(schemes=["argon2"], deprecated="auto")


def hash_password(plain: str) -> str:
    return _pwd.hash(plain)


def verify_password(plain: str, hashed: str) -> bool:
    return _pwd.verify(plain, hashed)
```

### 7.3 Depends — get_current_user

```python
# app/deps.py
from typing import Annotated, AsyncGenerator
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, ExpiredSignatureError
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select

from app.db import get_db
from app.domain.user import User
from app.services.jwt_util import decode_token
from app.errors import UnauthorizedException, NotFoundException


# tokenUrl = login endpoint 경로 (Swagger UI 의 "Authorize" 에 표시)
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
```

> **`Annotated[X, Depends(...)]`** — pydantic v2 / FastAPI 0.95+ 권장 형식 (PEP 593). IDE 가 type 추론 더 정확.
> **`SessionDep`, `CurrentUserDep`** — 자주 쓰는 dependency 를 alias 로 정리.

### 7.4 auth router

```python
# app/api/auth.py
from fastapi import APIRouter, status
from sqlalchemy import select

from app.deps import SessionDep
from app.schemas.auth import SignupRequest, LoginRequest, TokenResponse, UserResponse
from app.domain.user import User
from app.services.password import hash_password, verify_password
from app.services.jwt_util import create_access_token, create_refresh_token
from app.errors import ConflictException, UnauthorizedException

router = APIRouter()


@router.post("/signup", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def signup(payload: SignupRequest, db: SessionDep) -> User:
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
    return user


@router.post("/login", response_model=TokenResponse)
async def login(payload: LoginRequest, db: SessionDep) -> TokenResponse:
    user = await db.scalar(select(User).where(User.email == payload.email))
    if user is None or not verify_password(payload.password, user.password_hash):
        raise UnauthorizedException("이메일 또는 비밀번호가 올바르지 않습니다.")

    return TokenResponse(
        accessToken=create_access_token(
            subject=str(user.id),
            additional={"email": user.email, "name": user.name},
        ),
        refreshToken=create_refresh_token(subject=str(user.id)),
    )
```

> **`response_model=UserResponse`** — FastAPI 가 자동 직렬화 + OpenAPI schema 노출.
> **`db.scalar(select(...))`** — SQLAlchemy 2.x async — 단건 반환.
> **`await db.commit() / await db.refresh(user)`** — async 필수.

---

## 8. CRUD endpoint — 메모

### 8.1 service

```python
# app/services/memo_service.py
from typing import Sequence
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.domain.memo import Memo
from app.errors import NotFoundException, ForbiddenException


async def create_memo(db: AsyncSession, owner_id: int, title: str, content: str) -> Memo:
    memo = Memo(owner_id=owner_id, title=title, content=content)
    db.add(memo)
    await db.commit()
    await db.refresh(memo)
    return memo


async def list_memos(db: AsyncSession, owner_id: int, limit: int = 20, offset: int = 0) -> Sequence[Memo]:
    stmt = (
        select(Memo)
        .where(Memo.owner_id == owner_id)
        .order_by(Memo.created_at.desc())
        .limit(limit)
        .offset(offset)
    )
    result = await db.scalars(stmt)
    return result.all()


async def get_owned_memo(db: AsyncSession, memo_id: int, owner_id: int) -> Memo:
    memo = await db.get(Memo, memo_id)
    if memo is None:
        raise NotFoundException("메모")
    if memo.owner_id != owner_id:
        raise ForbiddenException("본인의 메모만 조회 가능합니다.")
    return memo


async def update_memo(db: AsyncSession, memo_id: int, owner_id: int, patch: dict) -> Memo:
    memo = await get_owned_memo(db, memo_id, owner_id)
    for key, value in patch.items():
        if value is not None and hasattr(memo, key):
            setattr(memo, key, value)
    await db.commit()
    await db.refresh(memo)
    return memo


async def delete_memo(db: AsyncSession, memo_id: int, owner_id: int) -> None:
    memo = await get_owned_memo(db, memo_id, owner_id)
    await db.delete(memo)
    await db.commit()
```

### 8.2 router

```python
# app/api/memos.py
from fastapi import APIRouter, status, Query

from app.deps import SessionDep, CurrentUserDep
from app.schemas.memo import (
    CreateMemoRequest, UpdateMemoRequest, MemoResponse, MemoListResponse,
)
from app.services import memo_service

router = APIRouter()


@router.get("", response_model=MemoListResponse)
async def list_memos(
    db: SessionDep,
    current_user: CurrentUserDep,
    limit: int = Query(20, ge=1, le=100),
    offset: int = Query(0, ge=0),
) -> dict:
    memos = await memo_service.list_memos(db, current_user.id, limit, offset)
    return {"items": memos, "total": len(memos)}


@router.post("", response_model=MemoResponse, status_code=status.HTTP_201_CREATED)
async def create_memo(
    payload: CreateMemoRequest,
    db: SessionDep,
    current_user: CurrentUserDep,
):
    return await memo_service.create_memo(db, current_user.id, payload.title, payload.content)


@router.get("/{memo_id}", response_model=MemoResponse)
async def get_memo(
    memo_id: int,
    db: SessionDep,
    current_user: CurrentUserDep,
):
    return await memo_service.get_owned_memo(db, memo_id, current_user.id)


@router.patch("/{memo_id}", response_model=MemoResponse)
async def update_memo(
    memo_id: int,
    payload: UpdateMemoRequest,
    db: SessionDep,
    current_user: CurrentUserDep,
):
    patch = payload.model_dump(exclude_unset=True)       # 명시한 필드만
    return await memo_service.update_memo(db, memo_id, current_user.id, patch)


@router.delete("/{memo_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_memo(
    memo_id: int,
    db: SessionDep,
    current_user: CurrentUserDep,
):
    await memo_service.delete_memo(db, memo_id, current_user.id)
```

> **`Query(20, ge=1, le=100)`** — query param 의 validation 도 FastAPI 가 자동.
> **`payload.model_dump(exclude_unset=True)`** — pydantic v2 — PATCH 시 명시한 필드만 추출.

### 8.3 실습

```bash
fastapi dev app/main.py

# 1. signup
curl -X POST http://localhost:8000/api/v1/auth/signup \
  -H "Content-Type: application/json" \
  -d '{"email":"yuchan@example.com","password":"P@ssw0rd1!","name":"yuchan"}'

# 2. login → access token
TOKEN=$(curl -sX POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"yuchan@example.com","password":"P@ssw0rd1!"}' \
  | jq -r .accessToken)

# 3. 메모 생성
curl -X POST http://localhost:8000/api/v1/memos \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"첫 메모","content":"안녕하세요"}'

# 4. Swagger UI 에서도 시험 가능
open http://localhost:8000/docs
# "Authorize" 클릭 → token 입력 → 모든 endpoint 시험
```

---

## 9. 테스트 — pytest + httpx

### 9.1 의존성

```bash
pip install pytest pytest-asyncio httpx aiosqlite
```

### 9.2 conftest.py

```python
# tests/conftest.py
import asyncio
from typing import AsyncGenerator
import pytest
import pytest_asyncio
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import async_sessionmaker, create_async_engine

from app.main import create_app
from app.db import Base, get_db


TEST_DATABASE_URL = "sqlite+aiosqlite:///:memory:"
test_engine = create_async_engine(TEST_DATABASE_URL)
TestSessionLocal = async_sessionmaker(test_engine, expire_on_commit=False)


@pytest_asyncio.fixture
async def db():
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    async with TestSessionLocal() as session:
        yield session
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)


@pytest_asyncio.fixture
async def client(db) -> AsyncGenerator[AsyncClient, None]:
    app = create_app()

    async def override_get_db():
        yield db

    app.dependency_overrides[get_db] = override_get_db

    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as ac:
        yield ac


@pytest_asyncio.fixture
async def auth_headers(client) -> dict:
    await client.post("/api/v1/auth/signup", json={
        "email": "t@example.com", "password": "P@ssw0rd1!", "name": "t",
    })
    res = await client.post("/api/v1/auth/login", json={
        "email": "t@example.com", "password": "P@ssw0rd1!",
    })
    token = res.json()["accessToken"]
    return {"Authorization": f"Bearer {token}"}
```

### 9.3 테스트

```python
# tests/test_memos.py
import pytest


@pytest.mark.asyncio
async def test_create_and_list_memo(client, auth_headers):
    res = await client.post("/api/v1/memos",
                            json={"title": "t1", "content": "c1"},
                            headers=auth_headers)
    assert res.status_code == 201
    memo_id = res.json()["id"]

    res = await client.get("/api/v1/memos", headers=auth_headers)
    assert res.status_code == 200
    body = res.json()
    assert body["total"] == 1
    assert body["items"][0]["id"] == memo_id


@pytest.mark.asyncio
async def test_cannot_access_others_memo(client, auth_headers):
    await client.post("/api/v1/auth/signup", json={
        "email": "other@example.com", "password": "P@ssw0rd1!", "name": "o",
    })
    res = await client.post("/api/v1/auth/login", json={
        "email": "other@example.com", "password": "P@ssw0rd1!",
    })
    other_token = res.json()["accessToken"]
    other_headers = {"Authorization": f"Bearer {other_token}"}

    res = await client.post("/api/v1/memos",
                            json={"title": "secret", "content": "..."},
                            headers=other_headers)
    other_memo_id = res.json()["id"]

    res = await client.get(f"/api/v1/memos/{other_memo_id}", headers=auth_headers)
    assert res.status_code == 403


@pytest.mark.asyncio
async def test_validation_error(client, auth_headers):
    res = await client.post("/api/v1/memos",
                            json={"content": "title 누락"},
                            headers=auth_headers)
    assert res.status_code == 422
    body = res.json()
    assert body["error"]["code"] == "VALIDATION_ERROR"
    assert "title" in body["error"]["details"]
```

```python
# pytest.ini or pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

```bash
pytest -v
# tests/test_memos.py::test_create_and_list_memo PASSED
# tests/test_memos.py::test_cannot_access_others_memo PASSED
# tests/test_memos.py::test_validation_error PASSED
```

---

## 10. 배포 — uvicorn + Docker

### 10.1 uvicorn — production

```bash
# dev (auto-reload)
fastapi dev app/main.py

# production (multi-worker)
uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4
# 또는 gunicorn + uvicorn worker
gunicorn -k uvicorn.workers.UvicornWorker -w 4 -b 0.0.0.0:8000 app.main:app
```

> **multi-worker**: 각 worker = 독립 process. `(CPU * 2) + 1` 시작점. event loop 가 worker 마다 별도.

### 10.2 Dockerfile

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

ENV PORT=8000
EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

### 10.3 docker-compose

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
      JWT_SECRET_KEY: dev-jwt
      DATABASE_URL: postgresql+asyncpg://memo:memo@db:5432/memo
    ports: ["8000:8000"]
    depends_on: [db]
    command: >
      sh -c "alembic upgrade head &&
             uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4"

volumes:
  dbdata:
```

### 10.4 운영 체크리스트

- [ ] `Settings` 의 `SECRET_KEY` / `JWT_SECRET_KEY` 가 vault 주입
- [ ] HTTPS (Nginx / ALB)
- [ ] `uvicorn --workers N` 가 인스턴스 spec 에 맞게 (`(CPU*2)+1`)
- [ ] PG `max_connections` ≥ workers × pool_size
- [ ] `alembic upgrade head` 가 컨테이너 startup
- [ ] log → stdout
- [ ] healthcheck endpoint
- [ ] `fastapi dev` 또는 `--reload` 운영 금지

```python
# app/main.py 에 healthcheck
@app.get("/healthz")
async def healthz():
    return {"status": "ok"}
```

---

## 11. 자주 막히는 함정

### 함정 1 — `async def` 안에서 sync DB / 외부 IO
이벤트 loop block → 전체 worker 정지. **async client (asyncpg / httpx) 만** 또는 `run_in_executor` 로 thread pool.

### 함정 2 — `Depends(get_db)` 가 async generator
일반 함수 `def get_db()` 면 안 됨. **`async def get_db() -> AsyncGenerator[AsyncSession, None]: async with ... as session: yield session`** 정확히.

### 함정 3 — `OAuth2PasswordBearer(auto_error=True)` (default)
토큰 없을 때 FastAPI 가 자동 `{"detail": "Not authenticated"}` 반환 → 우리 envelope 깨짐. **`auto_error=False`** + custom dependency 에서 `UnauthorizedException`.

### 함정 4 — pydantic v1 / v2 코드 혼동
- v1: `User.dict()`, `BaseModel.parse_obj`, `Config` 클래스
- v2: `User.model_dump()`, `BaseModel.model_validate`, `model_config = ConfigDict(...)`
**v2 기준** 본 가이드. 옛 튜토리얼 주의.

### 함정 5 — `python-jose` 의 clock skew
default 0초 — 1초 차이로 fail. `options={"leeway": 30}` 명시.

### 함정 6 — `from_attributes=True` 누락
ORM 객체 → pydantic 직접 변환 시 `ValidationError`. **`model_config = {"from_attributes": True}`** (v2).

### 함정 7 — SQLAlchemy 2.x 의 `lazy="select"` 와 async 비호환
async session 에서 lazy load = `MissingGreenlet`. **`lazy="selectin"`** 또는 `selectinload(...)` 명시.

### 함정 8 — Alembic 의 async template
`alembic init -t async migrations` (async template) 안 쓰면 sync engine 으로 시도 → 에러. **반드시 `-t async`**.

### 함정 9 — `Settings()` 가 환경변수 missing 시 import crash
docker 가 무한 restart → kubectl logs 만으론 원인 어려움. **`os.environ.get` + default** 또는 startup health check.

### 함정 10 — `uvicorn --reload` 운영 사용
dev only. file watcher 가 메모리 / CPU 잡아먹음. **production 은 `--workers N`** 만.

### 함정 11 — `BaseHTTPMiddleware` 가 streaming response 와 호환성 이슈
큰 파일 다운로드 / Server-Sent Events 에서 `BaseHTTPMiddleware` 사용 시 메모리 폭발. **raw ASGI middleware** (Starlette 의 `Middleware` 인터페이스) 사용.

### 함정 12 — multi-worker 마다 별도 cache / DB pool
process 분리 — `lru_cache` / dict cache 가 worker 별. **Redis / DB 같은 외부 저장소** 사용.

---

## 12. 다음 학습 로드맵

| 단계 | 학습 | vault 노트 |
| --- | --- | --- |
| 1 | 본 가이드 직접 따라 만들기 | (이 노트) |
| 2 | 응답 envelope 통일 + RequestId 추적 | [[common/response-envelope]] |
| 3 | 더 강한 보안 (OAuth2 / scopes / 권한) | [[common/security-config]] |
| 4 | SQLAlchemy async 깊이 + N+1 회피 | [[database/database]] (예정) |
| 5 | Redis 캐시 / rate-limit / distributed lock | [[api-design/cache-redis]] / [[api-design/rate-limiting]] (예정) |
| 6 | 파일 업로드 (S3) | [[api-design/file-upload-s3]] (예정) |
| 7 | 비동기 작업 (Celery 또는 FastAPI BackgroundTasks) | [[api-design/webhook-send]] (예정) |
| 8 | WebSocket (FastAPI native) | [[api-design/websocket-stomp]] (예정) |
| 9 | 운영 / Prometheus / Sentry | (예정) |

---

## 13. 참고 / cross-link

- [[python-fastapi|↑ FastAPI area]]
- [[../python-flask/getting-started|↗ Flask 가이드]] — 비교
- [[../python-django/getting-started]] (예정 — Django 변형)
- [[common/response-envelope]] — 응답 envelope 표준
- [[common/security-config]] — JWT / OAuth2 / Depends
- FastAPI 공식: <https://fastapi.tiangolo.com/>
- pydantic v2: <https://docs.pydantic.dev/latest/>
- SQLAlchemy 2.x async: <https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html>
- Alembic: <https://alembic.sqlalchemy.org/>
- python-jose: <https://python-jose.readthedocs.io/>
- passlib: <https://passlib.readthedocs.io/>
