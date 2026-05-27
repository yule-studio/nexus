---
title: "공통 Response Envelope (ApiEnvelope) — FastAPI"
kind: guideline
status: standard
created_at: 2026-05-27T00:00:00+09:00
agent: engineering-agent/tech-lead
tags: [api-design, response, envelope, python-fastapi, standard]
---

# 공통 Response Envelope (ApiEnvelope) — FastAPI

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/common-response-envelope|↗ Java 원본]]** · **[[../../python-flask/api-design/common-response-envelope|↗ Flask 변형]]**

> 본거지는 [[../common/response-envelope]] (`CommonResponse[T]`).
> 본 노트는 `bkurs-be` 스타일의 `ApiEnvelope[T]` 변형 — pydantic + RequestId middleware.

---

## 1-2-4. 왜 envelope / shape / ErrorCode

(Java 원본 [[../../spring/java-spring/api-design/common-response-envelope#1|§1, §2, §4]] 와 동일)

---

## 3. 컴포넌트 구조 (FastAPI)

```
app/common/
  ├── api/
  │   └── envelope.py            # ApiEnvelope (pydantic), ApiMeta, ApiEnvelopes
  ├── error/
  │   └── api_error.py           # ApiError pydantic
  ├── enums/
  │   └── error_code.py
  ├── exception/
  │   └── business.py            # BusinessException + 6 sub
  ├── handler/
  │   └── global_handler.py      # @app.exception_handler
  └── middleware/
      └── request_id.py          # X-Request-Id ASGI middleware
```

---

## 4. ApiEnvelope (pydantic)

```python
from __future__ import annotations
from datetime import datetime, timezone
from typing import Generic, TypeVar, Optional
from pydantic import BaseModel, ConfigDict

from .api_error import ApiError

T = TypeVar("T")


class ApiMeta(BaseModel):
    requestId: str
    timestamp: str


class ApiEnvelope(BaseModel, Generic[T]):
    model_config = ConfigDict(exclude_none=True)

    success: bool
    data: Optional[T] = None
    error: Optional[ApiError] = None
    meta: ApiMeta


def now_iso() -> str:
    return datetime.now(timezone.utc).isoformat()
```

> **pydantic v2 Generic[T]** — FastAPI 가 OpenAPI 에 schema 자동 노출.

---

## 5. RequestId middleware

```python
# app/middleware/request_id.py
import uuid
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response


class RequestIdMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next) -> Response:
        request_id = request.headers.get("X-Request-Id") or f"req-{uuid.uuid4().hex[:8]}"
        request.state.request_id = request_id
        response = await call_next(request)
        response.headers["X-Request-Id"] = request_id
        return response


# app/main.py
from .middleware.request_id import RequestIdMiddleware

app.add_middleware(RequestIdMiddleware)
```

> **FastAPI/Starlette idiom**: `BaseHTTPMiddleware.dispatch` async. `request.state` 가 per-request 저장소.

---

## 6. Endpoint 사용

```python
from app.common.api.envelope import ApiEnvelope, ApiMeta, now_iso


@router.get("/students/me", response_model=ApiEnvelope[StudentProfileView])
async def get_profile(
    request: Request,
    user: CurrentUserDep,
    service: Annotated[StudentProfileService, Depends(get_service)],
) -> ApiEnvelope[StudentProfileView]:
    data = await service.get(user.id)
    return ApiEnvelope(
        success=True,
        data=data,
        error=None,
        meta=ApiMeta(requestId=request.state.request_id, timestamp=now_iso()),
    )
```

> **`response_model=ApiEnvelope[X]`** — OpenAPI schema 자동 노출.

---

## 7. GlobalExceptionHandler

(Flask 변형 [[../../python-flask/api-design/common-response-envelope#6|§6]] 와 같은 패턴이지만 async)

```python
@app.exception_handler(BusinessException)
async def handle_business(request: Request, ex: BusinessException) -> JSONResponse:
    envelope = ApiEnvelope[None](
        success=False,
        data=None,
        error=ApiError(code=ex.code, status=ex.status, message=ex.message,
                       requestId=request.state.request_id),
        meta=ApiMeta(requestId=request.state.request_id, timestamp=now_iso()),
    )
    return JSONResponse(
        status_code=ex.status,
        content=envelope.model_dump(exclude_none=True),
    )
```

---

## 8. FE 처리 패턴

(Java 원본 [[../../spring/java-spring/api-design/common-response-envelope#8 FE 처리|§8]] 와 동일 TypeScript 코드)

---

## 9. 안티 패턴

(Java 원본 §10 그대로 + Python/FastAPI 추가)

### FastAPI 안티 패턴

- ❌ `response_model=ApiEnvelope` (generic 누락) — `data` 가 `Any` 로 schema 부정확
- ❌ middleware 안에서 `request.state` 외 다른 저장소 사용 (`g` 가 FastAPI 에 없음)
- ❌ Starlette 의 `BaseHTTPMiddleware` 가 streaming response 와 호환성 이슈 — 큰 파일은 raw ASGI middleware

---

## 10. 관련

- [[../common/response-envelope]] — 본거지
- [[../../spring/java-spring/api-design/common-response-envelope|↗ Java 원본]]
- [[../../python-flask/api-design/common-response-envelope|↗ Flask 변형]]
- [[api-design|↑ hub]]
