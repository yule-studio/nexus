---
title: "Rate Limiting (slowapi) — FastAPI"
kind: knowledge
status: current
created_at: 2026-05-26T23:45:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, api-design, recipe, rate-limit, slowapi]
---

# Rate Limiting (slowapi) — FastAPI

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/rate-limiting|↗ Java 원본]]** · **[[../../python-flask/api-design/rate-limiting|↗ Flask 변형]]**

> `slowapi` (Flask-Limiter 의 FastAPI 포팅) + Redis backend.

---

## 1. 무엇을 / 1.1 보호 대상 / 1.2 알고리즘

(Java 원본 [[../../spring/java-spring/api-design/rate-limiting#1|§1]] 와 동일)

---

## 2-3. 의존성 + 설정

```bash
pip install slowapi
```

```python
# app/extensions/limiter.py
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(
    key_func=get_remote_address,
    storage_uri="redis://localhost:6379",
    strategy="moving-window",
    default_limits=["1000/hour"],
    headers_enabled=True,
)
```

```python
# app/main.py
from slowapi.errors import RateLimitExceeded
from slowapi.middleware import SlowAPIMiddleware
from app.extensions.limiter import limiter


def create_app() -> FastAPI:
    app = FastAPI(...)
    app.state.limiter = limiter
    app.add_middleware(SlowAPIMiddleware)
    return app
```

---

## 4. 엔드포인트 단

```python
from app.extensions.limiter import limiter


@router.post("/auth/login")
@limiter.limit("5/minute")
async def login(request: Request, payload: LoginRequest):
    # ...
    pass


# user 기반 (JWT identity)
def user_or_ip(request: Request) -> str:
    # FastAPI 의 request.state.user 가 있다면
    user = getattr(request.state, "user", None)
    if user:
        return f"user:{user.id}"
    return get_remote_address(request)


@router.post("/payments/confirm")
@limiter.limit("10/minute", key_func=user_or_ip)
async def confirm_payment(request: Request, payload: ConfirmRequest):
    # ...
    pass
```

> **slowapi 한계**: decorator 가 `request: Request` 첫 인자 강제 (FastAPI 권장 형식과 다름). 명시적 `Request` 파라미터.

---

## 5. 429 envelope 통일

```python
# app/errors.py 에
from slowapi.errors import RateLimitExceeded


@app.exception_handler(RateLimitExceeded)
async def handle_rate_limit(request: Request, exc: RateLimitExceeded):
    body = CommonResponse.fail(
        ResponseCode.RATE_LIMIT_EXCEEDED,
        "요청이 너무 많습니다. 잠시 후 다시 시도해 주세요.",
    )
    return JSONResponse(
        status_code=429,
        content=body.model_dump(exclude_none=True),
        headers={"Retry-After": "60"},
    )
```

---

## 6. user 역할 별 가변 정책

```python
def dynamic_limit_fn(request: Request) -> str:
    user = getattr(request.state, "user", None)
    if user:
        if user.role == "ADMIN":
            return "1000/minute"
        if user.role == "SELLER":
            return "100/minute"
    return "60/minute"


@router.get("/items")
@limiter.limit(dynamic_limit_fn)
async def items(request: Request):
    # ...
    pass
```

---

## 7. Gateway / Nginx 단

(Java 원본 [[../../spring/java-spring/api-design/rate-limiting#4.3 Gateway / Nginx 단|§4.3]] 동일 — Nginx / API Gateway 가 1차 방어)

---

## 8. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/rate-limiting#8 함정 모음|§8]] 의 10 함정 그대로)

### FastAPI 한정

### 함정 11 — `Request` 인자 없으면 slowapi 동작 X
`@limiter.limit(...)` decorator 는 함수 signature 에 `request: Request` 필수. **FastAPI Depends 와 같이 명시**.

### 함정 12 — multi-worker (`uvicorn --workers N`) + in-memory storage
worker 마다 별도 카운터. **`storage_uri="redis://..."` 강제**.

### 함정 13 — `headers_enabled=True` 가 모든 endpoint 응답에 `X-RateLimit-*` 헤더
일부 health check 응답에도 — `/healthz` 같은 endpoint 는 `@limiter.exempt`.

---

## 9. 운영 체크리스트

(Java 원본 §9 + Python)

- [ ] storage_uri Redis
- [ ] @app.exception_handler(RateLimitExceeded) envelope
- [ ] /healthz 같은 endpoint `@limiter.exempt`
- [ ] X-Forwarded-For 신뢰 처리 (trusted proxy)

---

## 10. 관련

- [[../../spring/java-spring/api-design/rate-limiting|↗ Java 원본]]
- [[../../python-flask/api-design/rate-limiting|↗ Flask 변형]]
- [[distributed-lock]]
- [[../common/security-config]]
- [[api-design|↑ hub]]
