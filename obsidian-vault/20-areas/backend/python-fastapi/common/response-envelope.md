---
title: "표준 응답 envelope + 글로벌 예외 처리 (FastAPI)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T16:25:00+09:00
tags:
  - backend
  - python-fastapi
  - common
  - response
  - exception
  - production
---

# 표준 응답 envelope + 글로벌 예외 처리 (FastAPI)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + FastAPI exception_handler + pydantic BaseModel |

**[[../python-fastapi|↑ FastAPI]]** · **[[../../spring/java-spring/common/response-envelope|↗ Java 원본]]** · **[[../python-flask/common/response-envelope|↗ Flask 변형]]**

> FastAPI 진영의 envelope 표준. 외부 계약 (JSON envelope, ResponseCode) 은 Java/Kotlin/Flask 와 동일, 내부는 pydantic + async + dependency injection.

---

## 1. 왜 envelope 통일 / 2. 4 컴포넌트

(Java 원본 [[../../spring/java-spring/common/response-envelope#1|§1-§2]] 와 동일)

---

## 3. `CommonResponse` — pydantic BaseModel

```python
# app/common/dto.py
from __future__ import annotations
from typing import Generic, TypeVar, Optional
from pydantic import BaseModel, ConfigDict
from pydantic.generics import GenericModel  # pydantic v1 의 경우. v2 는 BaseModel 자체 Generic

from .enums import ResponseCode

T = TypeVar("T")


class CommonResponse(BaseModel, Generic[T]):
    """공통 응답 envelope. pydantic v2 표준."""

    model_config = ConfigDict(
        # result is None 이면 직렬화 제외 — Java @JsonInclude(NON_NULL) 동등
        exclude_none=True,
    )

    code: str
    status: str
    message: str
    result: Optional[T] = None

    @classmethod
    def success(cls, code: ResponseCode, message: str, data: Optional[T] = None) -> "CommonResponse[T]":
        return cls(code=code.code, status=code.name, message=message, result=data)

    @classmethod
    def fail(cls, code: ResponseCode, custom_message: Optional[str] = None) -> "CommonResponse[T]":
        return cls(
            code=code.code,
            status=code.name,
            message=custom_message or code.message,
        )
```

> **Python/pydantic idiom 포인트**:
> - `BaseModel + Generic[T]` — pydantic v2 가 직접 지원 (v1 은 `GenericModel`).
> - `model_config = ConfigDict(exclude_none=True)` — None 필드 직렬화 제외.
> - **FastAPI 가 자동 직렬화** — return value 가 `CommonResponse[Foo]` 면 OpenAPI schema 도 생성.

### 3.1 응답 예시

```json
{ "code": "OK_001", "status": "OK", "message": "회원가입 성공" }
```

```json
{
  "code": "OK_001", "status": "OK", "message": "로그인 성공",
  "result": { "accessToken": "eyJ...", "refreshToken": "eyJ..." }
}
```

---

## 4. `ResponseCode` enum

(Flask 변형 [[../python-flask/common/response-envelope#4|§4]] 와 동일 — Python Enum 표준)

```python
# app/common/enums.py
from enum import Enum
from http import HTTPStatus


class ResponseCode(Enum):
    OK                       = (HTTPStatus.OK,                       "OK_001",          "성공")
    # ... (전체 enum 은 Flask 변형 참조)
    INTERNAL_SERVER_ERROR    = (HTTPStatus.INTERNAL_SERVER_ERROR,    "INTERNAL_ERR_001", "서버 내부 오류")

    def __init__(self, http_status: HTTPStatus, code: str, message: str) -> None:
        self.http_status = http_status
        self.code = code
        self.message = message

    @property
    def is_5xx(self) -> bool:
        return 500 <= self.http_status.value < 600
```

---

## 5. `BusinessException`

```python
# app/common/exceptions.py
from typing import Optional
from .enums import ResponseCode


class BusinessException(Exception):
    def __init__(self, response_code: ResponseCode, custom_message: Optional[str] = None) -> None:
        self.response_code = response_code
        self.custom_message = custom_message or response_code.message
        super().__init__(self.custom_message)

    @property
    def is_5xx(self) -> bool:
        return self.response_code.is_5xx
```

---

## 6. FastAPI `exception_handler` — 글로벌

```python
# app/common/error_handlers.py
import logging
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from starlette.exceptions import HTTPException as StarletteHTTPException

from .dto import CommonResponse
from .enums import ResponseCode
from .exceptions import BusinessException

log = logging.getLogger(__name__)
EXCEPTION_ATTR = "apiLog.exception"


def register_error_handlers(app: FastAPI) -> None:

    @app.exception_handler(BusinessException)
    async def handle_business(request: Request, ex: BusinessException) -> JSONResponse:
        request.state.exception = ex
        if ex.is_5xx:
            log.error("BIZ 5xx: %s", ex, exc_info=True)
        else:
            log.warning("BIZ: code=%s, msg=%s", ex.response_code.code, ex.custom_message)

        body = CommonResponse.fail(ex.response_code, ex.custom_message)
        return JSONResponse(
            status_code=ex.response_code.http_status.value,
            content=body.model_dump(exclude_none=True),
        )

    @app.exception_handler(RequestValidationError)
    async def handle_validation(request: Request, ex: RequestValidationError) -> JSONResponse:
        request.state.exception = ex
        log.warning("VALIDATION: %s", ex.errors())

        # field 별 error 맵 — pydantic v2 error format
        field_errors = {
            ".".join(str(loc) for loc in err["loc"][1:] or err["loc"]): err["msg"]
            for err in ex.errors()
        }
        body = CommonResponse[dict](
            code=ResponseCode.INVALID_INPUT_FORMAT.code,
            status=ResponseCode.INVALID_INPUT_FORMAT.name,
            message="입력 형식이 올바르지 않습니다.",
            result=field_errors,
        )
        return JSONResponse(status_code=422, content=body.model_dump(exclude_none=True))

    @app.exception_handler(StarletteHTTPException)
    async def handle_http(request: Request, ex: StarletteHTTPException) -> JSONResponse:
        request.state.exception = ex
        log.warning("HTTP %s: %s", ex.status_code, ex.detail)

        code = ResponseCode.NOT_FOUND if ex.status_code == 404 else ResponseCode.BAD_REQUEST
        body = CommonResponse.fail(code, str(ex.detail) if ex.detail else "HTTP error")
        return JSONResponse(status_code=ex.status_code, content=body.model_dump(exclude_none=True))

    @app.exception_handler(Exception)
    async def handle_any(request: Request, ex: Exception) -> JSONResponse:
        request.state.exception = ex
        log.exception("UNCAUGHT")

        body = CommonResponse.fail(
            ResponseCode.INTERNAL_SERVER_ERROR,
            "서버 내부 오류가 발생했습니다.",
        )
        return JSONResponse(status_code=500, content=body.model_dump(exclude_none=True))
```

> **FastAPI idiom 포인트**:
> - `@app.exception_handler(Cls)` async function — Flask `errorhandler` 의 async 버전.
> - `request.state.exception` — Starlette 의 per-request 저장소 (Flask `request.environ` 동등).
> - `JSONResponse(status_code, content)` — Spring `ResponseEntity` 동등.
> - `RequestValidationError` (FastAPI) — pydantic validation 실패 자동.

---

## 7. Endpoint 에서 사용 패턴

### 7.1 직접 반환

```python
from fastapi import APIRouter
from app.common.dto import CommonResponse
from app.common.enums import ResponseCode

router = APIRouter(prefix="/api/v1/auth", tags=["auth"])


@router.post("/signup", response_model=CommonResponse[SignupResponse])
async def signup(payload: SignupRequest) -> CommonResponse[SignupResponse]:
    data = await signup_service.signup(payload)
    return CommonResponse.success(ResponseCode.OK, "회원가입 성공", data)
```

> **장점**: `response_model=CommonResponse[SignupResponse]` 가 OpenAPI schema 에 자동 노출 — Swagger UI 에서 정확한 응답 구조 미리 보기.

### 7.2 status code 지정

```python
from fastapi import status


@router.post(
    "/signup",
    response_model=CommonResponse[SignupResponse],
    status_code=status.HTTP_201_CREATED,
)
async def signup(payload: SignupRequest) -> CommonResponse[SignupResponse]:
    data = await signup_service.signup(payload)
    return CommonResponse.success(ResponseCode.OK, "회원가입 성공", data)
```

→ 실패는 throw `BusinessException` → handler 가 status 매핑. Endpoint 가 간결.

---

## 8. App 에 등록

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI

from .common.error_handlers import register_error_handlers
from .common.logging_middleware import ApiLogMiddleware


@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup
    yield
    # shutdown


app = FastAPI(
    title="Shop API",
    version="1.0.0",
    lifespan=lifespan,
)

register_error_handlers(app)
app.add_middleware(ApiLogMiddleware)

from .api.auth import router as auth_router
app.include_router(auth_router)
```

---

## 9. ApiLogMiddleware

```python
# app/common/logging_middleware.py
import time
import logging
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response

log = logging.getLogger("api_log")


class ApiLogMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next) -> Response:
        start = time.monotonic()
        ex = None
        try:
            response = await call_next(request)
            return response
        except Exception as e:
            ex = e
            raise
        finally:
            dur_ms = int((time.monotonic() - start) * 1000)
            # request.state.exception 은 exception_handler 가 설정
            captured = ex or getattr(request.state, "exception", None)
            log.info(
                "method=%s path=%s dur_ms=%d ex_type=%s ex_msg=%s",
                request.method, request.url.path, dur_ms,
                type(captured).__name__ if captured else None,
                str(captured)[:500] if captured else None,
            )
```

> **FastAPI/Starlette idiom**: `BaseHTTPMiddleware.dispatch(request, call_next)` async — Spring filter 의 async 버전.

---

## 10. 테스트

```python
# tests/test_envelope.py
import pytest
from fastapi.testclient import TestClient
from app.main import app
from app.common.exceptions import BusinessException
from app.common.enums import ResponseCode


@pytest.fixture
def client():
    return TestClient(app)


def test_business_exception_returns_correct_code(client):
    @app.get("/__test/biz")
    async def biz_endpoint():
        raise BusinessException(ResponseCode.DUPLICATE_DATA, "이미 가입")

    res = client.get("/__test/biz")
    assert res.status_code == 400
    body = res.json()
    assert body["code"] == "BADREQ_004"
    assert body["message"] == "이미 가입"


def test_validation_error_returns_422(client):
    class Payload(BaseModel):
        email: EmailStr

    @app.post("/__test/validate")
    async def validate_endpoint(p: Payload):
        return {"ok": True}

    res = client.post("/__test/validate", json={"email": "not-an-email"})
    assert res.status_code == 422
    body = res.json()
    assert "email" in body["result"]
```

---

## 11. 운영 체크리스트

- [ ] 모든 endpoint 에 try/except 없음 — `exception_handler` 가 처리
- [ ] 모든 도메인 예외가 `BusinessException(ResponseCode.X, ...)`
- [ ] `Exception` 캐치올은 운영에서 메시지 노출 X
- [ ] `RequestValidationError` 가 field-level error 응답
- [ ] 5xx 비율 모니터링 (Prometheus FastAPI exporter)
- [ ] `response_model=CommonResponse[X]` 명시 — OpenAPI schema 정확
- [ ] `ApiLogMiddleware` 가 모든 요청 기록
- [ ] uvicorn / hypercorn worker 수 + DB pool 매칭

---

## 12. 함정 모음 (FastAPI + Python 한정)

### 함정 1 — `ex.__str__()` 을 응답 메시지로
스택 / DB 패스워드 노출. **응답에는 일반 메시지**.

### 함정 2 — `JSONResponse(content=body)` 대신 `body.dict()` 직접 반환
FastAPI 가 자동 직렬화 — 그러나 `exception_handler` 에서는 `JSONResponse` 명시 필수 (return type 강제).

### 함정 3 — pydantic v1 ↔ v2 호환
- v1: `body.dict(exclude_none=True)`, `GenericModel`
- v2: `body.model_dump(exclude_none=True)`, `BaseModel` 자체 Generic
**버전 명시 + lock**.

### 함정 4 — `response_model=CommonResponse` (generic 없음)
type parameter 누락 시 `result` 가 `Any` 로 직렬화 — OpenAPI schema 부정확. **`CommonResponse[SignupResponse]`** 명시.

### 함정 5 — async exception_handler 에서 sync DB 호출
이벤트 loop block. async 호출 또는 `run_in_executor`.

### 함정 6 — `RequestValidationError` 의 `loc` 첫 element 가 "body" / "query"
field path 추출 시 `err["loc"][1:]` 로 skip — body field 만 보고 싶을 때.

### 함정 7 — Python 한정: pydantic Generic + `from __future__ import annotations`
forward reference 가 generic 매개변수 해석 못 해 런타임 에러 가능. v2 에선 대부분 해결.

### 함정 8 — Python 한정: `Optional[T]` vs `T | None` 혼용
pydantic v2 는 둘 다 OK. Python 3.10+ `T | None` 권장.

### 함정 9 — Python 한정: `request.state.x` 가 typed X
`request.state` 는 그냥 namespace — IDE / mypy 가 type 추론 못 함. 명시적 type hint 또는 wrapper class.

### 함정 10 — Python 한정: middleware 가 `BaseHTTPMiddleware` 인가 ASGI raw 인가
`BaseHTTPMiddleware` = 편리하지만 streaming response 와 호환성 이슈. 고성능 / streaming 은 raw ASGI middleware.

---

## 13. 관련

- [[security-config]] (다음) — OAuth2PasswordBearer + Depends
- [[../../spring/java-spring/common/response-envelope|↗ Java 원본]]
- [[../python-flask/common/response-envelope|↗ Flask 변형]]
- [[../api-design/api-design|↗ API 레시피]]
