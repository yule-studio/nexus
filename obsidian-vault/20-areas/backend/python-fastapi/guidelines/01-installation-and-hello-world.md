---
title: "01. 설치 + venv + Hello FastAPI"
kind: guide
status: current
created_at: 2026-05-27T14:40:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, guidelines, installation, hello-world]
---

# 01. 설치 + venv + Hello FastAPI

**[[guidelines|↑ Hub]]** → 다음 [[02-project-structure]]

> Python 3.12 + venv + FastAPI + uvicorn 설치 → 첫 endpoint + 자동 Swagger UI.

---

## 1. Python 3.12 + venv

(Flask 변형 [[../../python-flask/guidelines/01-installation-and-hello-world#1 Python 3.12 설치|§1-§2]] 와 동일 — Python / pyenv / venv 표준)

```bash
mkdir fastapi-memo-api && cd fastapi-memo-api
python3.12 -m venv .venv
source .venv/bin/activate
```

---

## 2. FastAPI 설치

```bash
pip install --upgrade pip
pip install "fastapi[standard]"
pip freeze > requirements.txt
```

`fastapi[standard]` 가 포함:
- `fastapi`
- `uvicorn[standard]` — ASGI server
- `fastapi-cli` — `fastapi dev` / `fastapi run` 명령
- `python-multipart` — form / file upload
- `email-validator` — `EmailStr` 검증
- `jinja2` — template (선택)
- `httpx` — 테스트

---

## 3. Hello FastAPI

### 3.1 main.py

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

### 3.2 실행

```bash
# 방법 1 — fastapi CLI (0.110+)
fastapi dev main.py
# * Running on http://127.0.0.1:8000

# 방법 2 — uvicorn 직접
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

```bash
# 테스트
curl http://localhost:8000/
# {"message":"Hello, FastAPI!"}

curl http://localhost:8000/hello/yuchan
# {"message":"Hello, yuchan!"}
```

### 3.3 자동 Swagger UI

- **Swagger UI**: <http://localhost:8000/docs>
- **ReDoc**: <http://localhost:8000/redoc>
- **OpenAPI JSON**: <http://localhost:8000/openapi.json>

→ 별도 설정 없이 endpoint + schema 자동. FastAPI 의 킬러 기능.

---

## 4. 핵심 개념

### 4.1 `@app.get` / `@app.post` / ... — async decorator

```python
@app.get("/users/{id}")
async def get_user(id: int):                           # int 자동 변환 + validation
    return {"id": id}


@app.post("/users")
async def create_user(user: UserCreate):               # pydantic body 자동
    return user


@app.delete("/users/{id}", status_code=204)
async def delete_user(id: int):
    ...
```

> **`async def`** vs `def`: async 면 event loop 안 실행 (non-blocking). DB / 외부 IO 호출 시 async 필수.
>
> sync `def` 도 OK — FastAPI 가 자동으로 thread pool 실행 (CPU bound 코드는 sync 권장).

### 4.2 type hint = validation

```python
@app.get("/items")
async def items(
    q: str | None = None,                              # ?q=... — optional
    limit: int = 20,                                    # ?limit=... — int 변환
    skip: int = 0,
):
    return {"q": q, "limit": limit, "skip": skip}
```

```bash
curl "http://localhost:8000/items?q=foo&limit=5"
# {"q":"foo","limit":5,"skip":0}

curl "http://localhost:8000/items?limit=abc"           # int 변환 실패 → 422 자동
# {"detail":[{"type":"int_parsing",...}]}
```

→ **type hint 가 곧 query validation**. 별도 schema 없이도.

### 4.3 pydantic Body

```python
from pydantic import BaseModel, EmailStr, Field


class UserCreate(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8, max_length=64)
    name: str = Field(min_length=1, max_length=100)


@app.post("/users")
async def create(user: UserCreate):
    return {"email": user.email, "name": user.name}
```

```bash
curl -X POST http://localhost:8000/users \
  -H "Content-Type: application/json" \
  -d '{"email":"not-email","password":"123","name":""}'
# 422
# {
#   "detail": [
#     {"type": "value_error", "loc": ["body", "email"], "msg": "value is not a valid email address"},
#     {"type": "string_too_short", "loc": ["body", "password"], "msg": "String should have at least 8 characters"},
#     {"type": "string_too_short", "loc": ["body", "name"], "msg": "String should have at least 1 character"}
#   ]
# }
```

---

## 5. 응답 형식

```python
# 1. dict / pydantic — 자동 JSON
@app.get("/a")
async def a():
    return {"key": "value"}                            # 200 + JSON


# 2. response_model 명시 (권장)
@app.get("/b", response_model=Greeting)
async def b() -> Greeting:
    return Greeting(message="hi")                       # OpenAPI schema 자동


# 3. status_code 명시
@app.post("/c", status_code=201)
async def c():
    return {"created": True}


# 4. Response 객체 — header / status 동시
from fastapi import Response

@app.get("/d")
async def d(response: Response):
    response.headers["X-Custom"] = "value"
    return {"ok": True}


# 5. JSONResponse — exception handler 등에서
from fastapi.responses import JSONResponse

@app.get("/e")
async def e():
    return JSONResponse(status_code=200, content={"key": "value"})
```

---

## 6. 자주 막히는 함정

### 함정 1 — sync `def` 안 blocking IO
event loop block → 성능 폭락. **DB / HTTP 호출은 async**.

### 함정 2 — `pip install fastapi` 만
uvicorn 누락 → `fastapi dev` / `uvicorn` 명령 못 씀. **`fastapi[standard]`**.

### 함정 3 — pydantic v1 / v2 혼동
v1: `.dict()`, `.parse_obj`, `Config` 클래스
v2: `.model_dump()`, `.model_validate`, `model_config = ConfigDict(...)`
**v2 기준** (FastAPI 0.100+ default).

### 함정 4 — `EmailStr` 사용 시 `email-validator` 누락
ImportError. `fastapi[standard]` 가 포함하지만 minimal install 시 `pip install email-validator`.

### 함정 5 — `uvicorn --reload` 운영 사용
file watcher = CPU. dev only. 운영은 `--workers N`.

### 함정 6 — `fastapi dev` vs `uvicorn`
`fastapi dev` = fastapi-cli (dev mode). 운영 `fastapi run --workers N` 또는 gunicorn + UvicornWorker.

### 함정 7 — `response_model` 누락
OpenAPI schema 가 부정확 (`Any`). **항상 명시**.

### 함정 8 — `model_dump()` vs `dict()`
v1 는 `dict()`, v2 는 `model_dump()`. 옛 코드 / 튜토리얼 주의.

---

## 7. 다음

✅ 설치 + Hello + 자동 /docs 끝.

→ [[02-project-structure]] — 본격 프로젝트 + pydantic-settings + lifespan
