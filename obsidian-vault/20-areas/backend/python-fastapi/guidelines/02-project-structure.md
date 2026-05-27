---
title: "02. 프로젝트 구조 + pydantic-settings + lifespan"
kind: guide
status: current
created_at: 2026-05-27T14:50:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, guidelines, structure, settings, lifespan]
---

# 02. 프로젝트 구조 + pydantic-settings + lifespan

**[[guidelines|↑ Hub]]** ← [[01-installation-and-hello-world]] · → 다음 [[03-domain-and-sqlalchemy]]

> 메모 API 패키지 구조 + pydantic-settings (env / Config) + lifespan event (startup / shutdown) + Depends 기본.

---

## 1. 권장 디렉토리

```
fastapi-memo-api/
├── .venv/
├── .env                          # gitignore
├── .gitignore
├── requirements.txt
├── pyproject.toml                # (선택)
├── README.md
├── app/
│   ├── __init__.py
│   ├── main.py                   # FastAPI() + include_router + lifespan
│   ├── config.py                 # pydantic-settings
│   ├── deps.py                   # Depends 함수들 (DB / get_current_user / SessionDep 등)
│   ├── db.py                     # async engine + AsyncSessionLocal
│   ├── domain/                   # SQLAlchemy 2.x 모델
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── memo.py
│   ├── schemas/                  # pydantic request/response
│   │   ├── auth.py
│   │   └── memo.py
│   ├── api/                      # APIRouter (Flask Blueprint 동등)
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   ├── memos.py
│   │   └── health.py
│   ├── services/                 # business logic (async)
│   │   ├── auth_service.py
│   │   ├── memo_service.py
│   │   └── password.py
│   ├── errors.py                 # custom Exception + handler
│   └── middlewares/              # CORS / 로깅 / request id
├── alembic.ini
├── migrations/                   # Alembic
└── tests/
    ├── conftest.py
    └── test_*.py
```

---

## 2. requirements.txt

```txt
fastapi[standard]==0.110.0
uvicorn[standard]==0.27.1
SQLAlchemy[asyncio]==2.0.29
asyncpg==0.29.0
alembic==1.13.1
python-jose[cryptography]==3.3.0
passlib[argon2]==1.7.4
argon2-cffi==23.1.0
pydantic==2.6.4
pydantic-settings==2.2.1
python-dotenv==1.0.1
httpx==0.27.0
pytest==8.1.1
pytest-asyncio==0.23.6
aiosqlite==0.20.0
```

---

## 3. `.env` + Settings (pydantic-settings)

```bash
# .env (gitignore)
SECRET_KEY=dev-secret-change-in-prod
JWT_SECRET_KEY=dev-jwt-32-bytes-please-please-change
DATABASE_URL=postgresql+asyncpg://memo:memo@localhost:5432/memo
CORS_ORIGINS=http://localhost:3000
```

```python
# app/config.py
from datetime import timedelta
from typing import List
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",                       # .env 의 미지 변수 무시
    )

    SECRET_KEY: str
    JWT_SECRET_KEY: str
    JWT_ACCESS_TOKEN_EXPIRES: timedelta = timedelta(minutes=15)
    JWT_REFRESH_TOKEN_EXPIRES: timedelta = timedelta(days=14)
    JWT_ALGORITHM: str = "HS256"
    JWT_ISSUER: str = "memo-api"

    DATABASE_URL: str
    CORS_ORIGINS: List[str] = ["http://localhost:3000"]


settings = Settings()                          # module-level singleton
```

> **pydantic-settings v2 의 좋은 점**:
> - `.env` / 시스템 env 자동 로드
> - **type 변환 + 검증** (`timedelta`, `List[str]` 자동 parsing)
> - missing 변수면 **import time 에 `ValidationError`** — fail-fast

---

## 4. main.py — FastAPI app + lifespan

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from .config import settings
from .db import engine
from .errors import register_error_handlers
from .api import auth, memos, health


@asynccontextmanager
async def lifespan(app: FastAPI):
    """startup → yield → shutdown 순서."""
    # startup
    print("FastAPI starting...")
    yield
    # shutdown
    await engine.dispose()                     # async engine cleanup
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
    app.include_router(health.router, tags=["health"])

    return app


app = create_app()                              # 모듈 단 인스턴스 — uvicorn 이 import
```

> **`lifespan`** = Spring 의 `ApplicationListener<ContextRefreshedEvent>` 동등. startup / shutdown 한 곳.
> **`@asynccontextmanager`** = async with 사용 가능한 context manager 데코레이터.

---

## 5. APIRouter — Flask Blueprint 동등

```python
# app/api/auth.py
from fastapi import APIRouter

router = APIRouter()


@router.post("/signup")
async def signup():
    ...


@router.post("/login")
async def login():
    ...
```

```python
# main.py 에서
app.include_router(auth.router, prefix="/api/v1/auth", tags=["auth"])
```

→ URL = `/api/v1/auth/signup`. `tags` = Swagger UI 그룹.

### 5.1 router 단 prefix / dependencies

```python
router = APIRouter(
    prefix="/api/v1/memos",
    tags=["memos"],
    dependencies=[Depends(verify_api_key)],     # router 전체에 적용
    responses={404: {"description": "Not found"}},
)
```

---

## 6. Depends — DI 기본

FastAPI 의 핵심 — **`Depends`** 가 모든 의존성 주입.

```python
from fastapi import Depends
from typing import Annotated


async def get_db():
    async with AsyncSessionLocal() as session:
        yield session


SessionDep = Annotated[AsyncSession, Depends(get_db)]


@router.get("/memos")
async def list_memos(db: SessionDep) -> list[Memo]:
    return await memo_service.list_all(db)
```

> **`Annotated[X, Depends(...)]`** = FastAPI 0.95+ 권장 형식 (PEP 593). IDE / mypy 가 type 정확히 추론.

### 6.1 dependency chain

```python
async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
    db: SessionDep,
) -> User:
    # 토큰 검증 + user 조회
    ...


async def require_admin(
    user: Annotated[User, Depends(get_current_user)],
) -> User:
    if user.role != "ADMIN":
        raise ForbiddenException()
    return user


@router.delete("/admin/users/{id}")
async def admin_delete(
    id: int,
    admin: Annotated[User, Depends(require_admin)],     # 전체 chain 자동 resolve
):
    ...
```

→ FastAPI 가 `Depends` graph 분석 후 순서대로 호출. 같은 dependency 는 한 번만 (request scope cache).

### 6.2 sync vs async dependency

```python
def get_db_sync():                              # sync — thread pool 에서 실행
    ...

async def get_db_async():                        # async — event loop 에서
    ...
```

둘 다 OK. 단 sync 안 sleep / blocking IO 는 event loop block.

---

## 7. .gitignore

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
```

---

## 8. 실행

```bash
# dev
fastapi dev app/main.py

# 운영
fastapi run app/main.py --workers 4

# 또는 uvicorn
uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4

# 또는 gunicorn + UvicornWorker
gunicorn -k uvicorn.workers.UvicornWorker -w 4 -b 0.0.0.0:8000 app.main:app
```

---

## 9. 자주 막히는 함정

### 함정 1 — `Settings()` import-time crash
환경변수 미설정 시 `ValidationError`. docker 무한 restart → kubectl logs.
**default 값 또는 startup health check**:
```python
class Settings(BaseSettings):
    SECRET_KEY: str = "fallback-dev-key"           # default
```
또는 `.env.example` 으로 동료가 인지.

### 함정 2 — `lifespan` 안 sync code
`def lifespan(app)` 가 아니라 `async def + @asynccontextmanager`. 잘못 쓰면 동작 X.

### 함정 3 — `Depends` 의 generator session 누수
`yield` 후 cleanup 안 됨. `async def get_db(): async with ... as session: yield session` 패턴 정확히.

### 함정 4 — `Annotated` 없이 `db: AsyncSession = Depends(get_db)` 식
구식. v0.95+ 는 `Annotated[X, Depends(...)]` 권장 — IDE / mypy 친화.

### 함정 5 — `app = create_app()` 가 module-level
uvicorn 이 `app.main:app` 로 import — 항상 module-level instance 있어야.

### 함정 6 — pydantic-settings v1 vs v2
v1: `BaseSettings` from `pydantic`. v2: `pydantic-settings` 별도 패키지.

### 함정 7 — `extra="ignore"` 안 하면 fail
`.env` 에 unrelated 변수 (예: gunicorn 변수) 있으면 `ValidationError`. **`extra="ignore"`**.

### 함정 8 — `CORSMiddleware` 의 `allow_credentials=True` + `*` origin
브라우저가 거부. JWT + Bearer 면 `False`.

### 함정 9 — `include_router(prefix="/api")` vs router 자체 `prefix`
둘 다 적용 가능. 중복 (`/api/api`) 주의.

### 함정 10 — Python 한정: thread pool size
sync `def` endpoint 가 많으면 thread pool 부족. `--workers N × --limit-concurrency` 조정.

---

## 10. 다음

✅ 프로젝트 구조 + Settings + lifespan + Depends 끝.

→ [[03-domain-and-sqlalchemy]] — SQLAlchemy 2.x async + asyncpg + Alembic
