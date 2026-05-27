---
title: "Rate Limiting (Flask-Limiter + Redis) — Flask"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T17:15:00+09:00
tags: [backend, python-flask, api-design, recipe, rate-limit]
---

# Rate Limiting (Flask-Limiter + Redis) — Flask

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/rate-limiting|↗ Java 원본]]**

---

## 1. 무엇을 / 1.1 보호 대상 / 1.2 알고리즘

(Java 원본 [[../../spring/java-spring/api-design/rate-limiting#1|§1]] 와 동일)

---

## 2. 의존성

```python
pip install Flask-Limiter redis
```

---

## 3. 설정

```python
# app/__init__.py
from flask import Flask
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(
    key_func=get_remote_address,
    storage_uri="redis://localhost:6379",
    strategy="moving-window",                     # token bucket 동등
    default_limits=["1000/hour"],                  # 글로벌 fallback
    headers_enabled=True,                          # X-RateLimit-* / Retry-After 자동
)


def create_app() -> Flask:
    app = Flask(__name__)
    limiter.init_app(app)
    return app
```

---

## 4. 엔드포인트 단 적용

```python
from app import limiter
from flask_jwt_extended import get_jwt_identity


# IP 기반 — login brute force 방어
@auth_bp.post("/login")
@limiter.limit("5/minute", key_func=get_remote_address)
def login():
    # ...
    pass


# user 기반 — Authorization 헤더에서 user ID
@payment_bp.post("/confirm")
@limiter.limit(
    "10/minute",
    key_func=lambda: get_jwt_identity() or get_remote_address(),
)
@jwt_required()
def confirm_payment():
    # ...
    pass


# 동적 — user role 별
def _dynamic_limit():
    claims = get_jwt() or {}
    role = claims.get("role")
    if role == "ADMIN":
        return "1000/minute"
    if role == "SELLER":
        return "100/minute"
    return "60/minute"


@api_bp.get("/items")
@limiter.limit(_dynamic_limit)
def list_items():
    # ...
    pass
```

> **Flask-Limiter idiom**: `@limiter.limit("N/period")` decorator. `key_func` 로 차원 (IP / user / mixed) 결정.

---

## 5. 429 응답 envelope 통일

```python
# app/common/error_handlers.py 에 추가
from flask_limiter.errors import RateLimitExceeded


@app.errorhandler(RateLimitExceeded)
def handle_rate_limit(ex: RateLimitExceeded):
    request.environ[EXCEPTION_ATTR] = ex
    body = CommonResponse.fail(
        ResponseCode.RATE_LIMIT_EXCEEDED,
        "요청이 너무 많습니다. 잠시 후 다시 시도해 주세요.",
    )
    response = jsonify(body.to_dict())
    response.headers["Retry-After"] = str(int(ex.description.split()[0]) if ex.description else 60)
    return response, 429
```

> **함정**: Flask-Limiter 가 자동으로 429 + plain text 응답 — envelope 통일하려면 `errorhandler` 명시.

---

## 6. user 역할 별 가변 정책

```python
@app.route("/api/v1/items")
@limiter.limit(_dynamic_limit, key_func=lambda: get_jwt_identity() or get_remote_address())
def items():
    # ...
    pass
```

`_dynamic_limit` 가 `str` 반환 → Flask-Limiter 가 매 요청마다 평가.

---

## 7. before_request 가 JWT 보다 먼저

```python
# app/__init__.py
@app.before_request
def rate_limit_first():
    # Flask-Limiter 가 모든 endpoint 에 자동 적용 (default_limits)
    # JWT verify 전에 — DB 조회 비용 절약
    pass
```

→ Flask-Limiter 가 자동으로 `before_request` hook 에 등록. 별도 순서 강제 없음.

---

## 8. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/rate-limiting#8 함정 모음|§8]] 의 10 함정 그대로)

### Flask 한정

### 함정 11 — `key_func=get_remote_address` 가 LB 뒤에서 항상 같은 IP
모든 요청이 LB IP. **`get_ipaddr` (Flask-Limiter) 또는 `X-Forwarded-For` 명시**:
```python
from flask import request
def trusted_ip():
    return request.headers.get("X-Forwarded-For", request.remote_addr).split(",")[0].strip()
```

### 함정 12 — `default_limits` 가 모든 endpoint 에 적용
health check / static asset 도 limit → 모니터링 ping 막힘. **`@limiter.exempt`** 명시.

### 함정 13 — `storage_uri` 미설정 = in-memory
multi-worker 면 worker 마다 별도 카운터. **Redis 강제**.

---

## 9. 운영 체크리스트

(Java 원본 §9 그대로)

- [ ] `storage_uri="redis://..."` 명시
- [ ] `errorhandler(RateLimitExceeded)` 로 envelope 통일
- [ ] trusted proxy 가 X-Forwarded-For 적절히 처리
- [ ] health check `@limiter.exempt`
- [ ] `default_limits` 가 너무 빡빡하지 않게

---

## 10. 관련

- [[../../spring/java-spring/api-design/rate-limiting|↗ Java 원본]]
- [[distributed-lock]]
- [[../common/security-config]]
- [[api-design|↑ hub]]
