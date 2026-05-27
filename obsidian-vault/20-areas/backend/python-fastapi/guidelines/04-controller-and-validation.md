---
title: "04. APIRouter + pydantic + 예외 envelope"
kind: guide
status: current
created_at: 2026-05-27T15:15:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, guidelines, controller, validation, pydantic]
---

# 04. APIRouter + pydantic + 예외 envelope

**[[guidelines|↑ Hub]]** ← [[03-domain-and-sqlalchemy]] · → 다음 [[05-security-and-deployment]]

> APIRouter + pydantic v2 BaseModel + async + 예외 envelope.

---

## 1. 응답 envelope (선택)

FastAPI 는 직접 response model 반환 (envelope 안 씌워도 자연스러움). 그러나 일관성 / 추적성 위해 envelope 권장:

```python
# app/schemas/envelope.py
from typing import Generic, TypeVar, Optional
from pydantic import BaseModel, ConfigDict

T = TypeVar("T")


class ApiError(BaseModel):
    code: str
    message: str
    details: Optional[dict] = None


class ApiResponse(BaseModel, Generic[T]):
    model_config = ConfigDict(exclude_none=True)

    success: bool
    message: Optional[str] = None
    data: Optional[T] = None
    error: Optional[ApiError] = None

    @classmethod
    def ok(cls, data: Optional[T] = None, message: str = "성공") -> "ApiResponse[T]":
        return cls(success=True, message=message, data=data)

    @classmethod
    def fail(cls, code: str, message: str, details: Optional[dict] = None) -> "ApiResponse[T]":
        return cls(success=False, error=ApiError(code=code, message=message, details=details))
```

> **`Generic[T]` + pydantic v2** — `ApiResponse[MemoResponse]` 로 OpenAPI schema 정확.

---

## 2. Custom Exception

```python
# app/errors.py
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
```

---

## 3. exception_handler — 글로벌

```python
# app/errors.py 에 추가
import logging
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from starlette.exceptions import HTTPException as StarletteHTTPException

from app.schemas.envelope import ApiResponse

log = logging.getLogger(__name__)


def register_error_handlers(app: FastAPI) -> None:

    @app.exception_handler(BusinessException)
    async def handle_business(request: Request, ex: BusinessException) -> JSONResponse:
        if ex.status >= 500:
            log.error("BIZ 5xx: %s", ex, exc_info=True)
        else:
            log.warning("BIZ %s: %s", ex.code, ex.message)

        return JSONResponse(
            status_code=ex.status,
            content=ApiResponse.fail(ex.code, ex.message).model_dump(exclude_none=True),
        )

    @app.exception_handler(RequestValidationError)
    async def handle_validation(request: Request, ex: RequestValidationError) -> JSONResponse:
        log.warning("VALIDATION: %s", ex.errors())

        # field 별 error 맵 — pydantic v2 error format
        details = {}
        for err in ex.errors():
            field = ".".join(str(loc) for loc in err["loc"][1:]) or str(err["loc"][0])
            details[field] = err["msg"]

        return JSONResponse(
            status_code=422,
            content=ApiResponse.fail(
                "VALIDATION_ERROR", "입력 형식이 올바르지 않습니다.", details,
            ).model_dump(exclude_none=True),
        )

    @app.exception_handler(StarletteHTTPException)
    async def handle_http(request: Request, ex: StarletteHTTPException) -> JSONResponse:
        return JSONResponse(
            status_code=ex.status_code,
            content=ApiResponse.fail(
                "HTTP_ERROR", str(ex.detail) if ex.detail else "",
            ).model_dump(exclude_none=True),
        )

    @app.exception_handler(Exception)
    async def handle_any(request: Request, ex: Exception) -> JSONResponse:
        log.exception("UNCAUGHT")
        return JSONResponse(
            status_code=500,
            content=ApiResponse.fail(
                "INTERNAL_SERVER_ERROR", "서버 내부 오류가 발생했습니다.",
            ).model_dump(exclude_none=True),
        )
```

---

## 4. Pydantic schema

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
    model_config = ConfigDict(from_attributes=True)         # ORM → pydantic 자동

    id: int
    title: str
    content: str
    createdAt: datetime = Field(alias="created_at")
    updatedAt: datetime = Field(alias="updated_at")


class MemoListResponse(BaseModel):
    items: list[MemoResponse]
    total: int
```

> **`model_config = ConfigDict(from_attributes=True)`** = ORM 객체 → pydantic 자동 변환 (`MemoResponse.model_validate(memo)`).
> **`Field(alias="created_at")`** = DB column ↔ JSON field 이름 다를 때.

자주 쓰는 Field option:

| option | 의미 |
| --- | --- |
| `min_length` / `max_length` | 문자열 / 컬렉션 길이 |
| `ge` / `le` / `gt` / `lt` | 숫자 범위 |
| `pattern` | 정규식 |
| `default` / `default_factory` | default 값 |
| `alias` | JSON field name |
| `examples` | OpenAPI 예시 |
| `description` | OpenAPI 설명 |

---

## 5. APIRouter

```python
# app/api/memos.py
from typing import Annotated
from fastapi import APIRouter, Depends, Query, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.db import get_db
from app.deps import get_current_user
from app.domain.user import User
from app.schemas.memo import (
    CreateMemoRequest, UpdateMemoRequest, MemoResponse, MemoListResponse,
)
from app.schemas.envelope import ApiResponse
from app.services import memo_service

router = APIRouter()


SessionDep = Annotated[AsyncSession, Depends(get_db)]
CurrentUserDep = Annotated[User, Depends(get_current_user)]


@router.get("", response_model=ApiResponse[MemoListResponse])
async def list_memos(
    db: SessionDep,
    user: CurrentUserDep,
    limit: int = Query(20, ge=1, le=100),
    offset: int = Query(0, ge=0),
):
    memos = await memo_service.list_memos(db, user.id, limit, offset)
    return ApiResponse.ok(
        MemoListResponse(items=[MemoResponse.model_validate(m) for m in memos], total=len(memos)),
        "조회 성공",
    )


@router.post("", response_model=ApiResponse[MemoResponse], status_code=status.HTTP_201_CREATED)
async def create_memo(
    payload: CreateMemoRequest,
    db: SessionDep,
    user: CurrentUserDep,
):
    memo = await memo_service.create_memo(db, user.id, payload.title, payload.content)
    return ApiResponse.ok(MemoResponse.model_validate(memo), "생성 완료")


@router.get("/{memo_id}", response_model=ApiResponse[MemoResponse])
async def get_memo(
    memo_id: int,
    db: SessionDep,
    user: CurrentUserDep,
):
    memo = await memo_service.get_owned_memo(db, memo_id, user.id)
    return ApiResponse.ok(MemoResponse.model_validate(memo))


@router.patch("/{memo_id}", response_model=ApiResponse[MemoResponse])
async def update_memo(
    memo_id: int,
    payload: UpdateMemoRequest,
    db: SessionDep,
    user: CurrentUserDep,
):
    patch = payload.model_dump(exclude_unset=True)         # 명시한 필드만
    memo = await memo_service.update_memo(db, memo_id, user.id, patch)
    return ApiResponse.ok(MemoResponse.model_validate(memo), "수정 완료")


@router.delete("/{memo_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_memo(
    memo_id: int,
    db: SessionDep,
    user: CurrentUserDep,
):
    await memo_service.delete_memo(db, memo_id, user.id)
```

> **`payload.model_dump(exclude_unset=True)`** = pydantic v2 — request 에 명시한 필드만 dict 로. PATCH 의 partial update 에 핵심.

---

## 6. 응답 예시

```bash
# 1. 정상
curl -X POST http://localhost:8000/api/v1/memos \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"title":"첫 메모","content":"안녕하세요"}'

# 201
# {
#   "success": true,
#   "message": "생성 완료",
#   "data": {
#     "id": 1, "title": "첫 메모", "content": "안녕하세요",
#     "createdAt": "2026-05-27T...", "updatedAt": "2026-05-27T..."
#   }
# }


# 2. validation 실패
curl -X POST http://localhost:8000/api/v1/memos \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"content":"title 누락"}'

# 422
# {
#   "success": false,
#   "error": {
#     "code": "VALIDATION_ERROR",
#     "message": "입력 형식이 올바르지 않습니다.",
#     "details": { "title": "Field required" }
#   }
# }


# 3. 권한 없음 / 다른 user 의 메모
curl http://localhost:8000/api/v1/memos/999 \
  -H "Authorization: Bearer <other_token>"

# 403
# {
#   "success": false,
#   "error": { "code": "FORBIDDEN", "message": "본인의 메모만 조회 가능합니다." }
# }
```

---

## 7. 자주 막히는 함정

### 함정 1 — `payload.dict()` vs `model_dump()`
v1 = `dict()`, v2 = `model_dump()`. 옛 코드 주의.

### 함정 2 — `exclude_unset=True` 누락
PATCH 에서 모든 필드 dict 로 → 명시 안 한 필드까지 update. **PATCH 는 `exclude_unset=True`**.

### 함정 3 — `from_attributes=True` 누락
`MemoResponse.model_validate(memo)` 가 `ValidationError`. ConfigDict 에 추가.

### 함정 4 — `Field(alias="created_at")` 와 ORM attribute mismatch
ORM 속성은 `created_at`, JSON 필드는 `createdAt`. alias 사용 시 `populate_by_name` 추가:
```python
model_config = ConfigDict(from_attributes=True, populate_by_name=True)
```

### 함정 5 — `RequestValidationError.errors()` 의 loc 첫 element
`["body", "field"]` 또는 `["query", "field"]` 형식. 첫 요소가 source. field path 추출 시 `loc[1:]`.

### 함정 6 — `response_model=ApiResponse` (generic 누락)
data 가 `Any` 로 직렬화 → OpenAPI schema 부정확. **`ApiResponse[MemoResponse]`** 명시.

### 함정 7 — `Field(default=None)` vs `Field(default_factory=list)`
list / dict default 는 `default_factory=list` (instance 공유 회피).

### 함정 8 — exception_handler 안 sync DB
async handler 안 sync DB query = event loop block. async 만.

### 함정 9 — `pydantic.EmailStr` 사용 시 `email-validator` 누락
`fastapi[standard]` 포함이지만 minimal install 면 `pip install email-validator`.

### 함정 10 — handler 우선순위
구체적 (`BusinessException`) → 일반 (`Exception`). 거꾸로면 모든 예외가 generic handler 로.

---

## 8. 다음

✅ Controller + Validation 끝.

→ [[05-security-and-deployment]] — JWT + Argon2 + OAuth2PasswordBearer + Docker
