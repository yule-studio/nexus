---
title: "공통 Response Envelope (ApiEnvelope) 표준 — Flask"
kind: guideline
status: standard
created_at: 2026-05-26T17:30:00+09:00
agent: engineering-agent/tech-lead
tags: [api-design, response, envelope, python-flask, standard]
---

# 공통 Response Envelope 표준 — Flask

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/common-response-envelope|↗ Java 원본]]**

> 본 vault 의 본거지 envelope 은 [[../common/response-envelope]] (`CommonResponse[T]`).
> 본 노트는 `bkurs-be` 스타일의 `ApiEnvelope[T]` 변형 — `success/data/error/meta` 4 필드 + `X-Request-Id` 까지.

---

## 1-2. 왜 envelope / 핵심 invariant

(Java 원본 [[../../spring/java-spring/api-design/common-response-envelope#1|§1-§2]] 와 동일)

---

## 3. 컴포넌트 구조

```
app/common/
  ├── api/
  │   ├── envelope.py          # ApiEnvelope (dataclass), ApiMeta, ApiEnvelopes (factory)
  ├── error/
  │   ├── api_error.py         # ApiError dataclass, ErrorCodeSpec
  ├── enums/
  │   └── error_code.py        # ErrorCode Enum
  ├── exception/
  │   ├── business.py          # BusinessException + 6 sub
  ├── handler/
  │   └── global_handler.py    # @app.errorhandler
  └── web/
      └── request_id.py        # before_request → g.request_id, X-Request-Id 헤더
```

---

## 4. `ApiEnvelope` (dataclass)

```python
from __future__ import annotations
from dataclasses import dataclass, asdict
from datetime import datetime, timezone
from typing import Generic, TypeVar, Optional
from .api_error import ApiError

T = TypeVar("T")


@dataclass
class ApiMeta:
    requestId: str
    timestamp: str


@dataclass
class ApiEnvelope(Generic[T]):
    success: bool
    data: Optional[T]
    error: Optional[ApiError]
    meta: ApiMeta

    def to_dict(self) -> dict:
        return asdict(self)


def now_iso() -> str:
    return datetime.now(timezone.utc).isoformat()
```

---

## 5. `ApiError`

```python
from dataclasses import dataclass
from typing import Optional


@dataclass
class FieldErrorItem:
    field: str
    code: str
    message: str


@dataclass
class ApiError:
    code: str
    status: int
    message: str
    messageKey: str
    fieldErrors: Optional[list[FieldErrorItem]] = None
    requestId: Optional[str] = None
```

---

## 6. `ErrorCode` enum

`COM-4001` / `ACC-4040` / `PAY-4090` 등 — Java 원본 [[../../spring/java-spring/api-design/common-response-envelope#4 ErrorCode|§4]] 그대로.

---

## 7. Controller 사용

```python
from app.common.api.envelope import ApiEnvelope, ApiMeta, now_iso
from flask import jsonify, g


@bp.get("/students/me")
@jwt_required()
def get_profile():
    data = service.get(get_jwt_identity())
    env = ApiEnvelope(
        success=True,
        data=data.to_dict(),
        error=None,
        meta=ApiMeta(requestId=g.request_id, timestamp=now_iso()),
    )
    return jsonify(env.to_dict()), 200
```

---

## 8. RequestId before_request

```python
import uuid
from flask import g, request, after_this_request


@app.before_request
def assign_request_id():
    rid = request.headers.get("X-Request-Id") or f"req-{uuid.uuid4().hex[:8]}"
    g.request_id = rid


@app.after_request
def attach_request_id_header(response):
    response.headers["X-Request-Id"] = g.request_id
    return response
```

---

## 9. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/common-response-envelope#10 안티 패턴|§10]] 의 안티 패턴 + Python 한정:)

### 함정 — Python 한정: `g` 객체 thread-local
Flask `g` 는 request-thread-local. async (Quart) 와 혼용 시 깨질 수 있음. Flask 3.x sync 만 사용 시 안전.

### 함정 — `asdict` 가 nested dataclass 까지 재귀
`ApiError.fieldErrors` (List[FieldErrorItem]) 도 자동 변환. 단 enum 은 변환 안 됨 — `.name` / `.value` 명시.

---

## 10. 관련

- [[../common/response-envelope]] — 본거지
- [[../../spring/java-spring/api-design/common-response-envelope|↗ Java 원본]]
- [[api-design|↑ hub]]
