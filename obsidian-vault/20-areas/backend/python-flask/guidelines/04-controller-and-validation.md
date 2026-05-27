---
title: "04. Blueprint route + marshmallow + 예외 envelope"
kind: guide
status: current
created_at: 2026-05-27T12:45:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-flask, guidelines, controller, validation, marshmallow]
---

# 04. Blueprint route + marshmallow + 예외 envelope

**[[guidelines|↑ Hub]]** ← [[03-domain-and-sqlalchemy]] · → 다음 [[05-security-and-deployment]]

> 응답 envelope + custom Exception + `@app.errorhandler` 글로벌 + marshmallow 입력 검증.

---

## 1. 응답 envelope

```python
# app/common/responses.py
from typing import Any, Optional
from flask import jsonify


def ok(data: Optional[Any] = None, message: str = "성공", status: int = 200):
    body = {"success": True, "message": message}
    if data is not None:
        body["data"] = data
    return jsonify(body), status


def fail(code: str, message: str, status: int, details: Any = None):
    body = {"success": False, "error": {"code": code, "message": message}}
    if details is not None:
        body["error"]["details"] = details
    return jsonify(body), status
```

응답 예시:

```json
// 성공
{ "success": true, "message": "메모 작성 완료", "data": { "id": 1, ... } }

// 실패
{ "success": false, "error": { "code": "FORBIDDEN", "message": "..." } }
```

---

## 2. Custom Exception

```python
# app/common/exceptions.py
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

사용:

```python
memo = memo_repo.find_by_id(id)
if memo is None:
    raise NotFoundException("메모")
```

---

## 3. 글로벌 errorhandler

```python
# app/common/error_handlers.py
import logging
from typing import Any
from flask import Flask, jsonify, request
from werkzeug.exceptions import HTTPException
from marshmallow import ValidationError

from .exceptions import BusinessException

log = logging.getLogger(__name__)


def _envelope(status: int, code: str, message: str, details: Any = None) -> dict:
    body = {"success": False, "error": {"code": code, "message": message}}
    if details is not None:
        body["error"]["details"] = details
    return body


def register_error_handlers(app: Flask) -> None:

    @app.errorhandler(BusinessException)
    def handle_business(ex: BusinessException):
        if ex.status >= 500:
            log.error("BIZ 5xx: %s", ex, exc_info=True)
        else:
            log.warning("BIZ %s: %s", ex.code, ex.message)
        return jsonify(_envelope(ex.status, ex.code, ex.message)), ex.status

    @app.errorhandler(ValidationError)
    def handle_validation(ex: ValidationError):
        log.warning("VALIDATION: %s", ex.messages)
        return jsonify(_envelope(422, "VALIDATION_ERROR",
                                 "입력 형식이 올바르지 않습니다.", ex.messages)), 422

    @app.errorhandler(HTTPException)
    def handle_http(ex: HTTPException):
        # 404 / 405 / 415 등 werkzeug 표준 에러
        return jsonify(_envelope(ex.code or 500, "HTTP_ERROR", ex.description or "")), ex.code or 500

    @app.errorhandler(Exception)
    def handle_any(ex: Exception):
        log.exception("UNCAUGHT")
        return jsonify(_envelope(500, "INTERNAL_SERVER_ERROR",
                                 "서버 내부 오류가 발생했습니다.")), 500
```

`create_app(...)` 안에서:

```python
from .common.error_handlers import register_error_handlers
register_error_handlers(app)
```

---

## 4. marshmallow Schema — 입력 검증

```python
# app/schemas/memo.py
from marshmallow import Schema, fields, validate


class CreateMemoSchema(Schema):
    title = fields.String(required=True, validate=validate.Length(min=1, max=200))
    content = fields.String(load_default="", validate=validate.Length(max=10_000))


class UpdateMemoSchema(Schema):
    title = fields.String(validate=validate.Length(min=1, max=200))
    content = fields.String(validate=validate.Length(max=10_000))


class MemoResponseSchema(Schema):
    id = fields.Integer()
    title = fields.String()
    content = fields.String()
    createdAt = fields.DateTime(attribute="created_at")
    updatedAt = fields.DateTime(attribute="updated_at")


class MemoListQuerySchema(Schema):
    limit = fields.Integer(load_default=20, validate=validate.Range(min=1, max=100))
    offset = fields.Integer(load_default=0, validate=validate.Range(min=0))
```

> **`attribute="created_at"`** — DB column name (snake_case) ↔ JSON field name (camelCase) 매핑.
> **`load_default`** — 입력 없을 때 default (loading 시).

자주 쓰는 field:

| Field | 의미 |
| --- | --- |
| `String` | 문자열 |
| `Integer`, `Float`, `Boolean` | 기본 type |
| `Email` | email format |
| `URL` | URL format (schemes 명시 권장) |
| `DateTime` | ISO-8601 |
| `Date`, `Time` | 날짜만, 시간만 |
| `UUID` | UUID format |
| `List(fields.X)` | 리스트 |
| `Nested(Schema)` | 중첩 |
| `Method("method_name")` | 다른 method 호출 |
| `Function(lambda obj: ...)` | 함수 |

자주 쓰는 validate:

| validate | 의미 |
| --- | --- |
| `validate.Length(min=, max=)` | 길이 |
| `validate.Range(min=, max=)` | 숫자 범위 |
| `validate.OneOf([...])` | enum |
| `validate.Regexp(r"...")` | 정규식 |
| `validate.Email()` | email (또는 `fields.Email`) |

---

## 5. Blueprint route 작성

```python
# app/api/memos.py
from flask import Blueprint, request
from flask_jwt_extended import jwt_required, get_jwt_identity

from app.common.responses import ok
from app.schemas.memo import (
    CreateMemoSchema, UpdateMemoSchema, MemoResponseSchema, MemoListQuerySchema,
)
from app.services import memo_service

memos_bp = Blueprint("memos", __name__, url_prefix="/api/v1/memos")


@memos_bp.get("")
@jwt_required()
def list_memos():
    query = MemoListQuerySchema().load(request.args)
    user_id = int(get_jwt_identity())

    memos = memo_service.list_for_owner(user_id, query["limit"], query["offset"])
    return ok(MemoResponseSchema(many=True).dump(memos), "조회 성공")


@memos_bp.post("")
@jwt_required()
def create_memo():
    payload = CreateMemoSchema().load(request.get_json())
    user_id = int(get_jwt_identity())

    # owner 로딩은 service 안에서
    memo = memo_service.create(user_id, payload["title"], payload["content"])
    return ok(MemoResponseSchema().dump(memo), "생성 완료", status=201)


@memos_bp.get("/<int:memo_id>")
@jwt_required()
def get_memo(memo_id: int):
    user_id = int(get_jwt_identity())
    memo = memo_service.get_owned(memo_id, user_id)
    return ok(MemoResponseSchema().dump(memo))


@memos_bp.patch("/<int:memo_id>")
@jwt_required()
def update_memo(memo_id: int):
    payload = UpdateMemoSchema().load(request.get_json())
    user_id = int(get_jwt_identity())

    memo = memo_service.update_memo(memo_id, user_id, payload.get("title"), payload.get("content"))
    return ok(MemoResponseSchema().dump(memo), "수정 완료")


@memos_bp.delete("/<int:memo_id>")
@jwt_required()
def delete_memo(memo_id: int):
    user_id = int(get_jwt_identity())
    memo_service.delete_memo(memo_id, user_id)
    return ok(message="삭제 완료"), 204
```

> **`MemoListQuerySchema().load(request.args)`** — query string 검증 가능. `request.args` 는 `ImmutableMultiDict` 라 marshmallow 가 자동 처리.

---

## 6. 응답 예시

```bash
# 1. 정상 생성
curl -X POST http://localhost:5000/api/v1/memos \
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
curl -X POST http://localhost:5000/api/v1/memos \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"content":"title 누락"}'

# 422
# {
#   "success": false,
#   "error": {
#     "code": "VALIDATION_ERROR",
#     "message": "입력 형식이 올바르지 않습니다.",
#     "details": { "title": ["Missing data for required field."] }
#   }
# }


# 3. 권한 없음
curl http://localhost:5000/api/v1/memos/999 \
  -H "Authorization: Bearer <other_token>"

# 403
# {
#   "success": false,
#   "error": {
#     "code": "FORBIDDEN",
#     "message": "본인의 메모만 조회 가능합니다."
#   }
# }
```

---

## 7. flask-smorest — Blueprint 통합 (선택)

`flask-smorest` 는 marshmallow + Blueprint + Swagger 통합. 더 강력하지만 학습 필요. 자세한 건 [[06-swagger-openapi]] 참고.

```python
from flask_smorest import Blueprint, abort

memos_bp = Blueprint("memos", "memos", url_prefix="/api/v1/memos")


@memos_bp.route("/<int:memo_id>")
class MemoResource(MethodView):
    @memos_bp.response(200, MemoResponseSchema)
    @jwt_required()
    def get(self, memo_id: int):
        memo = memo_service.get_owned(memo_id, int(get_jwt_identity()))
        return memo                                # smorest 가 자동 직렬화
```

→ 본 가이드는 **기본 Flask + marshmallow** 로 진행. smorest 는 큰 프로젝트에 검토.

---

## 8. 자주 막히는 함정

### 함정 1 — `@jwt_required()` 누락
인증 없이 endpoint 호출 가능. 보호 endpoint 마다 명시.

### 함정 2 — `MemoSchema().load(request.get_json())` 가 None
`Content-Type: application/json` 헤더 없으면 `get_json()` 가 None. validation 도 None 으로 fail. **헤더 강제**.

### 함정 3 — `errorhandler(Exception)` 가 jsonify 안 함
template 렌더링 모드면 HTML 반환 → 클라가 JSON 기대하다 깨짐. **무조건 jsonify**.

### 함정 4 — Blueprint 별 errorhandler vs app 단
`@blueprint.errorhandler` 는 해당 BP 만. 전역은 `@app.errorhandler`. 본 가이드는 전역.

### 함정 5 — werkzeug HTTPException 처리 누락
404 / 405 등이 raw HTML 반환. envelope 으로 매핑.

### 함정 6 — Schema `attribute` 와 `data_key` 혼동
- `attribute="created_at"` — model field name
- `data_key="createdAt"` — JSON field name
**둘 다 사용**:
```python
createdAt = fields.DateTime(attribute="created_at", data_key="createdAt")
```

### 함정 7 — `Schema().dump(orm_object)` 의 attribute 못 찾음
relationship field 가 lazy 면 dump 시점에 query → N+1. **service 단에서 미리 로드**.

### 함정 8 — `ValidationError.messages` 가 dict 또는 list
nested schema 시 list. envelope details 에 그대로 노출 OK.

### 함정 9 — `request.args` 가 query string 만
form data 는 `request.form`. JSON body 는 `request.get_json()`. **혼동 X**.

### 함정 10 — `marshmallow` 3.x vs 4.x 호환
4.x 가 곧 출시 — `load_default` / `dump_default` 명명 변경 등. 본 가이드는 3.x 기준.

---

## 9. 다음

✅ Controller + Validation 끝.

→ [[05-security-and-deployment]] — Flask-JWT-Extended + Argon2 + gunicorn + Docker
