---
title: "표준 응답 envelope + 글로벌 예외 처리 (Flask)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T16:00:00+09:00
tags:
  - backend
  - python-flask
  - common
  - response
  - exception
  - production
---

# 표준 응답 envelope + 글로벌 예외 처리 (Flask)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + Flask `errorhandler` + dataclass envelope |

**[[../python-flask|↑ Flask]]** · **[[../../spring/java-spring/common/response-envelope|↗ Java 원본]]** · **[[../../spring/kotlin-spring/common/response-envelope|↗ Kotlin 변형]]**

> **이 문서가 본거지.** Flask app 의 모든 응답이 동일 envelope.
> 외부 계약 (JSON envelope, ResponseCode 체계) 은 Java/Kotlin 원본과 동일.

---

## 1. 왜 envelope 통일이 첫 작업인가

(Java 원본 [[../../spring/java-spring/common/response-envelope#1|§1]] 와 동일 — 도메인 무관)

---

## 2. 4 컴포넌트 한눈에

```
[Service] ── raise BusinessException(ResponseCode.X, "메시지") ──┐
                                                                  │
[View func] ── return ok(data, "msg") ────────────────────┐       │
                                                           ▼       ▼
                                              ┌────────────────────┐
                                              │ @app.errorhandler  │
                                              │   - BusinessException → CommonResponse(code+msg)
                                              │   - ValidationError (marshmallow / pydantic) → 422
                                              │   - HTTPException → 매핑
                                              │   - Exception (캐치올) → 500
                                              └────────────────────┘
                                                       │
                                                       ▼
                                       flask.Response(json, status, headers)
```

---

## 3. `CommonResponse` — 응답 envelope

```python
# app/common/dto.py
from __future__ import annotations
from dataclasses import dataclass, field, asdict
from typing import Generic, TypeVar, Optional, Any

from .enums import ResponseCode

T = TypeVar("T")


@dataclass
class CommonResponse(Generic[T]):
    code: str                       # "OK_001" / "BADREQ_002" — stable string
    status: str                     # enum name ("OK" / "UNAUTHORIZED" ...)
    message: str
    result: Optional[T] = None      # None 이면 직렬화 제외

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

    def to_dict(self) -> dict[str, Any]:
        d = asdict(self)
        # result is None 이면 제외 — @JsonInclude(NON_NULL) 동등
        if d.get("result") is None:
            d.pop("result", None)
        return d
```

> **Python idiom 포인트**:
> - `@dataclass(Generic[T])` — Java record + 제네릭 동등.
> - `@classmethod` factory — Kotlin `companion object` 와 비슷.
> - `result` 가 `None` 이면 JSON 에서 빼는 건 `to_dict` 시 직접.
> - `from __future__ import annotations` — forward reference 활성 (Python 3.10+ 권장).

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

```python
# app/common/enums.py
from enum import Enum
from http import HTTPStatus


class ResponseCode(Enum):
    # (http_status, code, message)
    OK                       = (HTTPStatus.OK,                       "OK_001",          "성공")
    OK_EMPTY                 = (HTTPStatus.OK,                       "OK_002",          "성공 + 데이터 없음")

    BAD_REQUEST              = (HTTPStatus.BAD_REQUEST,              "BADREQ_001",      "잘못된 요청입니다.")
    INVALID_INPUT_FORMAT     = (HTTPStatus.BAD_REQUEST,              "BADREQ_002",      "입력 형식이 올바르지 않습니다.")
    MISSING_REQUIRED_FIELD   = (HTTPStatus.BAD_REQUEST,              "BADREQ_003",      "필수 입력값이 누락되었습니다.")
    DUPLICATE_DATA           = (HTTPStatus.BAD_REQUEST,              "BADREQ_004",      "이미 존재하는 데이터입니다.")

    UNAUTHORIZED             = (HTTPStatus.UNAUTHORIZED,             "UNAUTH_001",      "인증이 필요합니다.")
    INVALID_TOKEN            = (HTTPStatus.UNAUTHORIZED,             "UNAUTH_002",      "유효하지 않은 토큰입니다.")

    FORBIDDEN                = (HTTPStatus.FORBIDDEN,                "FORBID_001",      "권한이 없습니다.")
    INSUFFICIENT_PERMISSION  = (HTTPStatus.FORBIDDEN,                "FORBID_002",      "해당 리소스에 접근할 권한이 없습니다.")

    NOT_FOUND                = (HTTPStatus.NOT_FOUND,                "NOTFND_001",      "요청한 리소스를 찾을 수 없습니다.")
    USER_NOT_FOUND           = (HTTPStatus.NOT_FOUND,                "NOTFND_002",      "사용자를 찾을 수 없습니다.")

    CONFLICT                 = (HTTPStatus.CONFLICT,                 "CONFLT_001",      "리소스 상태 충돌")

    EXPIRED_AUTH_CODE        = (HTTPStatus.GONE,                     "GONE_001",        "인증 코드가 만료되었습니다.")
    DELETED_RESOURCE         = (HTTPStatus.GONE,                     "GONE_002",        "이미 삭제된 리소스입니다.")
    USER_ALREADY_DELETED     = (HTTPStatus.GONE,                     "GONE_003",        "이미 삭제된 사용자입니다.")

    AUTH_UNVERIFIED          = (HTTPStatus.UNPROCESSABLE_ENTITY,     "UNPROC_001",      "인증이 완료되지 않았습니다.")
    ADDITIONAL_INFO_REQUIRED = (HTTPStatus.UNPROCESSABLE_ENTITY,     "UNPROC_002",      "추가 정보 입력이 필요합니다.")
    SOCIAL_EMAIL_MISSING     = (HTTPStatus.UNPROCESSABLE_ENTITY,     "UNPROC_003",      "소셜 로그인에서 이메일이 제공되지 않았습니다.")

    RATE_LIMIT_EXCEEDED      = (HTTPStatus.TOO_MANY_REQUESTS,        "RATE_001",        "요청이 너무 많습니다. 잠시 후 다시 시도해 주세요.")

    INTERNAL_SERVER_ERROR    = (HTTPStatus.INTERNAL_SERVER_ERROR,    "INTERNAL_ERR_001", "서버 내부 오류")
    DATABASE_ERROR           = (HTTPStatus.INTERNAL_SERVER_ERROR,    "INTERNAL_ERR_002", "데이터베이스 오류")
    EXTERNAL_API_ERROR       = (HTTPStatus.INTERNAL_SERVER_ERROR,    "INTERNAL_ERR_003", "외부 API 호출 중 오류")
    FILE_PROCESSING_ERROR    = (HTTPStatus.INTERNAL_SERVER_ERROR,    "INTERNAL_ERR_004", "파일 처리 오류")

    def __init__(self, http_status: HTTPStatus, code: str, message: str) -> None:
        self.http_status = http_status
        self.code = code
        self.message = message

    @property
    def is_5xx(self) -> bool:
        return 500 <= self.http_status.value < 600
```

> **Python idiom**:
> - `Enum` 의 `__init__` — 튜플 분해해서 멤버 attribute 로.
> - `@property` — 호출 없이 `code.is_5xx`.
> - `HTTPStatus` (stdlib) — 매직 넘버 회피.

---

## 5. `BusinessException`

```python
# app/common/exceptions.py
from typing import Optional
from .enums import ResponseCode


class BusinessException(Exception):
    """도메인 / 비즈니스 예외 베이스."""

    def __init__(self, response_code: ResponseCode, custom_message: Optional[str] = None) -> None:
        self.response_code = response_code
        self.custom_message = custom_message or response_code.message
        super().__init__(self.custom_message)

    @property
    def is_5xx(self) -> bool:
        return self.response_code.is_5xx
```

### 5.1 사용 예

```python
from app.common.exceptions import BusinessException
from app.common.enums import ResponseCode

# Service
if user_repo.exists_by_email_hash(email_hash):
    raise BusinessException(ResponseCode.DUPLICATE_DATA, "이미 가입된 이메일입니다.")

# not found
user = user_repo.find_by_id(user_id)
if user is None:
    raise BusinessException(ResponseCode.USER_NOT_FOUND)

# 권한
if not order.is_owned_by(current_user_id):
    raise BusinessException(ResponseCode.FORBIDDEN, "본인의 주문만 조회 가능합니다.")
```

---

## 6. `errorhandler` — 글로벌 예외 매핑

```python
# app/common/error_handlers.py
import logging
from flask import Flask, jsonify, request
from werkzeug.exceptions import HTTPException
from marshmallow import ValidationError

from .dto import CommonResponse
from .enums import ResponseCode
from .exceptions import BusinessException

log = logging.getLogger(__name__)

EXCEPTION_ATTR = "apiLog.exception"


def register_error_handlers(app: Flask) -> None:

    @app.errorhandler(BusinessException)
    def handle_business(ex: BusinessException):
        request.environ[EXCEPTION_ATTR] = ex
        if ex.is_5xx:
            log.error("BIZ 5xx: %s", ex, exc_info=True)
        else:
            log.warning("BIZ: code=%s, msg=%s", ex.response_code.code, ex.custom_message)

        body = CommonResponse.fail(ex.response_code, ex.custom_message)
        return jsonify(body.to_dict()), ex.response_code.http_status.value

    @app.errorhandler(ValidationError)
    def handle_validation(ex: ValidationError):
        request.environ[EXCEPTION_ATTR] = ex
        log.warning("VALIDATION: %s", ex.messages)

        body = CommonResponse(
            code=ResponseCode.INVALID_INPUT_FORMAT.code,
            status=ResponseCode.INVALID_INPUT_FORMAT.name,
            message="입력 형식이 올바르지 않습니다.",
            result=ex.messages,
        )
        return jsonify(body.to_dict()), 422

    @app.errorhandler(HTTPException)
    def handle_http(ex: HTTPException):
        # 404 / 405 / 415 등 werkzeug 가 던지는 표준 HTTP 에러
        request.environ[EXCEPTION_ATTR] = ex
        log.warning("HTTP %s: %s", ex.code, ex.description)

        code = ResponseCode.NOT_FOUND if ex.code == 404 else ResponseCode.BAD_REQUEST
        body = CommonResponse.fail(code, ex.description or "HTTP error")
        return jsonify(body.to_dict()), ex.code or 500

    @app.errorhandler(Exception)
    def handle_any(ex: Exception):
        request.environ[EXCEPTION_ATTR] = ex
        log.exception("UNCAUGHT")

        body = CommonResponse.fail(
            ResponseCode.INTERNAL_SERVER_ERROR,
            "서버 내부 오류가 발생했습니다.",
        )
        return jsonify(body.to_dict()), 500
```

> **Python/Flask idiom 포인트**:
> - `@app.errorhandler(Cls)` — Flask 가 예외 타입 → handler 매핑.
> - `request.environ[KEY]` — Spring `request.setAttribute` 동등.
> - `log.exception(...)` — `log.error(..., exc_info=True)` 단축.
> - `(jsonify, status)` 튜플 반환 — Flask 표준.

---

## 7. View function 에서 사용

```python
# app/api/auth.py
from flask import Blueprint, request
from app.common.dto import CommonResponse
from app.common.enums import ResponseCode

auth_bp = Blueprint("auth", __name__, url_prefix="/api/v1/auth")


@auth_bp.post("/signup")
def signup():
    payload = SignupRequestSchema().load(request.get_json())   # marshmallow ValidationError → 422
    data = signup_service.signup(payload)

    body = CommonResponse.success(ResponseCode.OK, "회원가입 성공", data)
    return jsonify(body.to_dict()), ResponseCode.OK.http_status.value
```

> **함수 단순화** — 헬퍼:

```python
# app/common/responses.py
from typing import Any, Optional
from flask import jsonify
from .dto import CommonResponse
from .enums import ResponseCode


def ok(message: str = "성공", data: Optional[Any] = None, code: ResponseCode = ResponseCode.OK):
    body = CommonResponse.success(code, message, data)
    return jsonify(body.to_dict()), code.http_status.value
```

```python
@auth_bp.post("/signup")
def signup():
    payload = SignupRequestSchema().load(request.get_json())
    return ok("회원가입 성공", signup_service.signup(payload))
```

---

## 8. App factory 에 등록

```python
# app/__init__.py
from flask import Flask
from .common.error_handlers import register_error_handlers
from .common.logging_filter import ApiLogMiddleware


def create_app() -> Flask:
    app = Flask(__name__)
    app.config.from_object("app.config.Config")

    register_error_handlers(app)
    app.wsgi_app = ApiLogMiddleware(app.wsgi_app)            # access log + 예외 메타

    from .api.auth import auth_bp
    app.register_blueprint(auth_bp)
    return app
```

---

## 9. ApiLogMiddleware — 모든 요청 추적

```python
# app/common/logging_filter.py
import time
import logging
from typing import Callable
from werkzeug.wrappers import Request

log = logging.getLogger("api_log")
EXCEPTION_ATTR = "apiLog.exception"


class ApiLogMiddleware:
    def __init__(self, wsgi_app: Callable) -> None:
        self.wsgi_app = wsgi_app

    def __call__(self, environ, start_response):
        start = time.monotonic()
        status_holder: list[str] = []

        def custom_start_response(status, headers, exc_info=None):
            status_holder.append(status)
            return start_response(status, headers, exc_info)

        try:
            return self.wsgi_app(environ, custom_start_response)
        finally:
            dur_ms = int((time.monotonic() - start) * 1000)
            ex = environ.get(EXCEPTION_ATTR)
            req = Request(environ)
            log.info(
                "method=%s path=%s status=%s dur_ms=%d ex_type=%s ex_msg=%s",
                req.method, req.path,
                status_holder[0] if status_holder else "0",
                dur_ms,
                type(ex).__name__ if ex else None,
                str(ex)[:500] if ex else None,
            )
```

---

## 10. 테스트

```python
# tests/test_response_envelope.py
import pytest
from app import create_app
from app.common.enums import ResponseCode
from app.common.exceptions import BusinessException


@pytest.fixture
def client():
    app = create_app()
    app.testing = True
    return app.test_client()


def test_business_exception_returns_correct_code(client):
    # 임시 endpoint
    @app.route("/__test/biz")
    def biz_endpoint():
        raise BusinessException(ResponseCode.DUPLICATE_DATA, "이미 가입")

    res = client.get("/__test/biz")
    assert res.status_code == 400
    body = res.get_json()
    assert body["code"] == "BADREQ_004"
    assert body["message"] == "이미 가입"


def test_uncaught_does_not_leak(client):
    @app.route("/__test/boom")
    def boom():
        raise RuntimeError("DB password = abc123")

    res = client.get("/__test/boom")
    assert res.status_code == 500
    assert res.get_json()["message"] == "서버 내부 오류가 발생했습니다."
    # 내부 메시지 노출 X
```

---

## 11. 운영 체크리스트

- [ ] 모든 view 에 try/except 없음 — 예외는 errorhandler 가
- [ ] 모든 도메인 예외가 `BusinessException(ResponseCode.X, ...)`
- [ ] `Exception` 캐치올은 운영에서 메시지 노출 X
- [ ] 5xx 비율 모니터링 (APM / Prometheus flask exporter)
- [ ] 새 `ResponseCode` 추가 시 클라이언트 측에 알림
- [ ] `ApiLogMiddleware` 가 모든 응답을 기록
- [ ] Marshmallow `ValidationError` 매핑 (또는 pydantic `ValidationError`)
- [ ] werkzeug `HTTPException` 매핑 (404 등)
- [ ] Flask blueprint 마다 errorhandler 가 잡히는지 통합 테스트

---

## 12. 함정 모음

### 함정 1 — `ex.__str__()` 을 응답 메시지로
스택 / DB 패스워드 / 내부 path 노출. **응답에는 일반 메시지, 디버그는 로그**.

### 함정 2 — `errorhandler(Exception)` 가 jsonify 안 함
template 렌더링 모드면 HTML 페이지 반환 → 클라가 JSON 기대하다 깨짐. **무조건 jsonify**.

### 함정 3 — Validation 의 details 없음
사용자가 어느 필드가 틀렸는지 모름. **`result` 필드에 `ex.messages`** (marshmallow).

### 함정 4 — Flask 의 `errorhandler` 가 blueprint 별로 분리됨
`@blueprint.errorhandler(...)` 만 등록하면 다른 blueprint 의 예외는 못 잡음. **`@app.errorhandler(...)`** 전역.

### 함정 5 — werkzeug `HTTPException` 미잡음
404 / 405 등 표준 에러도 envelope 으로 통일해야 클라가 한 가지 모양만 처리.

### 함정 6 — Python 한정: `dataclass` + `Generic[T]` 가 직렬화 시 type lost
`asdict(...)` 는 dataclass field 만 직렬화 — type parameter `T` 는 dict 변환 시 그냥 `Any`. 그래도 정상 동작 (런타임에 type erasure 없음).

### 함정 7 — Python 한정: `None` 과 JSON `null`
`result: None` 가 JSON 에 `null` 로 직렬화됨. `to_dict()` 에서 명시적으로 pop 해야 envelope 의 NON_NULL 정책 충족.

### 함정 8 — Python 한정: enum value 가 tuple
`ResponseCode.OK.value` = `(HTTPStatus.OK, "OK_001", "성공")` tuple. `.code` / `.message` / `.http_status` attribute 로 접근.

### 함정 9 — `request.environ[KEY]` 가 thread-safe X
WSGI 의 environ 은 request 마다 새로 — thread-local. 그러나 multi-worker (gunicorn) 환경에선 같은 environ 가 다른 worker 에 가지 않으니 안전.

### 함정 10 — `log.exception` 안에서 sensitive data
exception 메시지가 stack trace 에 비밀번호 / 토큰 포함 시 그대로 로그. **redact filter** 또는 sentry breadcrumb 검토.

---

## 13. 관련

- [[security-config]] (다음) — JWT + CORS + Flask-JWT-Extended
- [[../../spring/java-spring/common/response-envelope|↗ Java 원본]]
- [[../../spring/kotlin-spring/common/response-envelope|↗ Kotlin 변형]]
- [[../api-design/api-design|↗ API 레시피]]
