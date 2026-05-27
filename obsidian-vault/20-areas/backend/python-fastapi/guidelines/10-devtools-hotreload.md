---
title: "10. DevTools — uvicorn --reload + fastapi dev + pytest-asyncio"
kind: guide
status: current
created_at: 2026-05-27T16:40:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, guidelines, devtools, hot-reload, pytest]
---

# 10. DevTools — uvicorn --reload + fastapi dev + pytest-asyncio

**[[guidelines|↑ Hub]]** ← [[09-logging-mdc]]

> dev cycle 가속 + 테스트 + 코드 품질 도구.

---

## 1. fastapi dev — 권장 dev 명령

```bash
fastapi dev app/main.py
```

내부적으로 `uvicorn --reload` 호출 + dev 친화 default. FastAPI 0.110+.

---

## 2. uvicorn --reload (직접)

```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

| 옵션 | 의미 |
| --- | --- |
| `--reload` | source 변경 시 auto restart |
| `--reload-dir app` | watch 디렉토리 명시 |
| `--reload-include "*.py"` | watch 확장자 |
| `--reload-exclude "*.log"` | 제외 |
| `--workers N` | multi-process (--reload 와 호환 X) |
| `--host 0.0.0.0` | bind |
| `--port 8000` | port |
| `--log-config logging.yaml` | logging |
| `--access-log` | access log |
| `--limit-concurrency 1000` | 동시 처리 한계 |

> **`--reload` 운영 절대 X** — file watcher 가 CPU. 운영은 `--workers N`.

---

## 3. IDE 통합 — VS Code

`.vscode/launch.json`:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "FastAPI",
            "type": "debugpy",
            "request": "launch",
            "module": "uvicorn",
            "args": [
                "app.main:app",
                "--reload",
                "--host", "0.0.0.0",
                "--port", "8000"
            ],
            "jinja": true,
            "justMyCode": false,
            "env": {
                "PYTHONUNBUFFERED": "1"
            }
        }
    ]
}
```

→ VS Code 의 breakpoint 사용.

---

## 4. PyCharm 통합

`Run/Debug Configurations` → `+` → `FastAPI` 선택. `app file` 지정 후 실행.

→ PyCharm 의 breakpoint + uvicorn 자동.

---

## 5. pytest-asyncio — async 테스트

```bash
pip install pytest pytest-asyncio httpx aiosqlite
```

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


TEST_DB_URL = "sqlite+aiosqlite:///:memory:"
test_engine = create_async_engine(TEST_DB_URL)
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
async def auth_headers(client):
    await client.post("/api/v1/auth/signup", json={
        "email": "t@e.com", "password": "P@ssw0rd1!", "name": "t",
    })
    res = await client.post("/api/v1/auth/login", json={
        "email": "t@e.com", "password": "P@ssw0rd1!",
    })
    token = res.json()["data"]["accessToken"]
    return {"Authorization": f"Bearer {token}"}
```

```python
# tests/test_memos.py
import pytest


@pytest.mark.asyncio
async def test_create_memo(client, auth_headers):
    res = await client.post(
        "/api/v1/memos",
        json={"title": "t1", "content": "c1"},
        headers=auth_headers,
    )
    assert res.status_code == 201
    assert res.json()["data"]["title"] == "t1"


@pytest.mark.asyncio
async def test_unauthorized(client):
    res = await client.get("/api/v1/memos")
    assert res.status_code == 401
```

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"                            # 모든 async test 자동 인식
```

```bash
pytest -v
```

---

## 6. 코드 품질 — ruff / black / mypy

[[../../python-flask/guidelines/10-devtools-hotreload#9 코드 품질|Flask §9]] 와 동일:

```bash
pip install ruff black mypy
```

```bash
ruff check . --fix
black .
mypy app/
```

---

## 7. pre-commit hook

```bash
pip install pre-commit
```

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.10.0
    hooks:
      - id: mypy
        additional_dependencies: [pydantic, sqlalchemy]
```

```bash
pre-commit install
```

---

## 8. IPython / breakpoint

```bash
pip install ipython ipdb
```

```python
# 코드 안
breakpoint()                                     # PYTHONBREAKPOINT=ipdb.set_trace
# 또는
import ipdb; ipdb.set_trace()
```

→ 해당 라인에서 멈추고 ipython 같은 interactive console.

> **async 안 `breakpoint()`** = OK (Python 3.11+).

---

## 9. dependency_overrides — 테스트 / mock

```python
# tests/test_memos.py
def test_with_mock_user(client):
    from app.deps import get_current_user
    from app.domain.user import User

    async def mock_user():
        return User(id=1, email="mock@e.com", name="mock")

    client.app.dependency_overrides[get_current_user] = mock_user

    res = client.get("/api/v1/memos")
    assert res.status_code == 200
```

→ 모든 endpoint 의 `Depends(get_current_user)` 가 mock 반환.

---

## 10. HTTP Client — IntelliJ / VS Code 의 `.http` 파일

```http
### 1. signup
POST http://localhost:8000/api/v1/auth/signup
Content-Type: application/json

{"email":"yuchan@example.com","password":"P@ssw0rd1!","name":"yuchan"}

### 2. login
POST http://localhost:8000/api/v1/auth/login
Content-Type: application/json

{"email":"yuchan@example.com","password":"P@ssw0rd1!"}

> {% client.global.set("token", response.body.data.accessToken); %}

### 3. 메모 생성
POST http://localhost:8000/api/v1/memos
Authorization: Bearer {{token}}
Content-Type: application/json

{"title":"첫 메모","content":"안녕"}
```

→ git commit 가능. Postman 대체.

---

## 11. 자주 막히는 함정

### 함정 1 — `uvicorn --reload` 운영
file watcher = CPU. dev only.

### 함정 2 — `fastapi dev` 와 `uvicorn --reload` 혼동
거의 동등. `fastapi dev` 는 fastapi-cli 의 wrapper.

### 함정 3 — pytest-asyncio 의 `asyncio_mode`
default 가 `strict` → 매 test 에 `@pytest.mark.asyncio` 데코레이터. **`asyncio_mode = "auto"`** 권장.

### 함정 4 — `dependency_overrides` cleanup 누락
test 후 override 가 다음 test 에 영향. **각 test 후 `app.dependency_overrides.clear()`** 또는 fixture teardown.

### 함정 5 — VS Code 의 multiprocess debugging
`--workers N` + breakpoint = subprocess 디버깅 어려움. **dev 는 `--workers 1`**.

### 함정 6 — `breakpoint()` 가 production
PYTHONBREAKPOINT 가 ipdb 면 운영에서 멈춤. **`PYTHONBREAKPOINT=0`** 운영 env.

### 함정 7 — pytest-asyncio + SQLAlchemy async session 공유
test 간 session 격리 — fixture 마다 새 session.

### 함정 8 — uvicorn `--limit-concurrency` 미설정
무한 동시 연결 = 메모리 폭발. **명시 권장** (1000 ~ 10000).

---

## 12. 다음 학습 — 가이드 끝

✅ 10 단계 입문 + 기본 세팅 완주!

| 단계 | 노트 |
| --- | --- |
| 11 | [[../common/response-envelope]] |
| 12 | [[../common/security-config]] |
| 13 | [[../api-design/api-design]] |
| 14 | [[../database/jpa]] · [[../database/mybatis]] |

---

## 13. 참고

- [[guidelines|↑ Guidelines hub]]
- [[../python-fastapi|↑ FastAPI area]]
- FastAPI 공식: <https://fastapi.tiangolo.com/>
- pytest-asyncio: <https://pytest-asyncio.readthedocs.io/>
- structlog: <https://www.structlog.org/>
- asgi-correlation-id: <https://github.com/snok/asgi-correlation-id>
