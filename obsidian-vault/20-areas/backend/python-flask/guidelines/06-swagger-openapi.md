---
title: "06. Swagger / OpenAPI — flask-smorest 또는 apispec"
kind: guide
status: current
created_at: 2026-05-27T13:15:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-flask, guidelines, swagger, openapi, smorest, apispec]
---

# 06. Swagger / OpenAPI — flask-smorest 또는 apispec

**[[guidelines|↑ Hub]]** ← [[05-security-and-deployment]] · → 다음 [[07-controller-patterns]]

> Flask 의 Swagger 통합 옵션:
> - **flask-smorest** (권장) — marshmallow + Blueprint + Swagger 한 set. 강력하지만 학습 곡선.
> - **apispec** — marshmallow + 수동 spec 생성. 가볍지만 boilerplate.
> - **flask-swagger-ui** — UI 만, spec 은 직접.

본 가이드는 **flask-smorest** 위주.

---

## 1. flask-smorest 의존성

```bash
pip install flask-smorest
# 같이 설치되는 것: marshmallow, apispec, flask-swagger-ui
```

---

## 2. 설정

```python
# app/extensions.py 에 추가
from flask_smorest import Api

api = Api()
```

```python
# app/config.py 에 추가
class Config:
    # ...
    API_TITLE = "Memo API"
    API_VERSION = "v1"
    OPENAPI_VERSION = "3.0.3"
    OPENAPI_URL_PREFIX = "/api/docs"                  # base URL
    OPENAPI_SWAGGER_UI_PATH = "/swagger"              # → /api/docs/swagger
    OPENAPI_SWAGGER_UI_URL = "https://cdn.jsdelivr.net/npm/swagger-ui-dist/"
    OPENAPI_REDOC_PATH = "/redoc"                     # → /api/docs/redoc
    OPENAPI_REDOC_URL = "https://cdn.jsdelivr.net/npm/redoc@next/bundles/redoc.standalone.js"

    # JWT Bearer schema 자동 (manual 도 가능)
    API_SPEC_OPTIONS = {
        "security": [{"BearerAuth": []}],
        "components": {
            "securitySchemes": {
                "BearerAuth": {
                    "type": "http",
                    "scheme": "bearer",
                    "bearerFormat": "JWT",
                }
            }
        },
    }
```

```python
# app/__init__.py 의 create_app 안
from .extensions import api

def create_app(config_class=Config) -> Flask:
    app = Flask(__name__)
    app.config.from_object(config_class)
    # ...
    api.init_app(app)
    # ...
```

---

## 3. Blueprint 를 smorest 식으로

```python
# app/api/memos_smorest.py
from flask.views import MethodView
from flask_smorest import Blueprint, abort
from flask_jwt_extended import jwt_required, get_jwt_identity

from app.schemas.memo import (
    CreateMemoSchema, UpdateMemoSchema, MemoResponseSchema, MemoListQuerySchema,
)
from app.services import memo_service

memos_bp = Blueprint(
    "memos", "memos",
    url_prefix="/api/v1/memos",
    description="메모 CRUD",
)


@memos_bp.route("/")
class MemoListView(MethodView):

    @memos_bp.arguments(MemoListQuerySchema, location="query")
    @memos_bp.response(200, MemoResponseSchema(many=True))
    @jwt_required()
    def get(self, query):
        """내 메모 목록 (page / limit)."""
        user_id = int(get_jwt_identity())
        return memo_service.list_for_owner(user_id, query["limit"], query["offset"])

    @memos_bp.arguments(CreateMemoSchema)
    @memos_bp.response(201, MemoResponseSchema)
    @jwt_required()
    def post(self, payload):
        """메모 생성."""
        user_id = int(get_jwt_identity())
        return memo_service.create(user_id, payload["title"], payload["content"])


@memos_bp.route("/<int:memo_id>")
class MemoView(MethodView):

    @memos_bp.response(200, MemoResponseSchema)
    @jwt_required()
    def get(self, memo_id: int):
        """메모 단건 조회."""
        return memo_service.get_owned(memo_id, int(get_jwt_identity()))

    @memos_bp.arguments(UpdateMemoSchema)
    @memos_bp.response(200, MemoResponseSchema)
    @jwt_required()
    def patch(self, payload, memo_id: int):
        """메모 수정."""
        return memo_service.update_memo(
            memo_id, int(get_jwt_identity()),
            payload.get("title"), payload.get("content"),
        )

    @memos_bp.response(204)
    @jwt_required()
    def delete(self, memo_id: int):
        """메모 삭제."""
        memo_service.delete_memo(memo_id, int(get_jwt_identity()))
```

> **smorest 마법**:
> - `@memos_bp.arguments(Schema)` — request body 자동 검증 + 함수 인자로 주입.
> - `@memos_bp.response(200, Schema)` — response 자동 직렬화.
> - **docstring** → endpoint 설명 자동.

### 3.1 등록

```python
# app/__init__.py
api.register_blueprint(memos_bp)
# 또는 (flask 기본 + smorest 양립)
app.register_blueprint(memos_bp)
```

---

## 4. 첫 동작

```bash
flask run --debug
```

- **Swagger UI**: <http://localhost:5000/api/docs/swagger>
- **ReDoc**: <http://localhost:5000/api/docs/redoc>
- **OpenAPI JSON**: <http://localhost:5000/api/docs/openapi.json>

→ 모든 endpoint + schema 자동 노출.

---

## 5. JWT Bearer Authorize

Swagger UI 우측 상단 `Authorize` 버튼:
1. 클릭
2. `Bearer <token>` 입력 (Bearer 빼고 token 만 입력하는 형식도 있음 — UI 안내 따름)
3. 모든 endpoint 호출 시 자동 Authorization 헤더 추가

→ `Api Spec Options` 의 `securitySchemes` 가 핵심.

---

## 6. apispec 식 (대안 — flask-smorest 안 쓰고)

`apispec` + `apispec-webframeworks` 만:

```bash
pip install apispec apispec-webframeworks
```

```python
# app/openapi.py
from apispec import APISpec
from apispec.ext.marshmallow import MarshmallowPlugin
from apispec_webframeworks.flask import FlaskPlugin

spec = APISpec(
    title="Memo API",
    version="1.0.0",
    openapi_version="3.0.3",
    plugins=[FlaskPlugin(), MarshmallowPlugin()],
)


def register_spec(app: Flask) -> None:
    # endpoint 별 docstring 의 OpenAPI YAML 부분 파싱
    with app.test_request_context():
        for rule in app.url_map.iter_rules():
            view_fn = app.view_functions[rule.endpoint]
            spec.path(view=view_fn)

    @app.get("/api/docs/openapi.json")
    def openapi_json():
        return jsonify(spec.to_dict())
```

→ view function 의 docstring 안 OpenAPI YAML 작성. boilerplate 많음. **flask-smorest 권장**.

---

## 7. Schema 에 OpenAPI 메타 추가

```python
from marshmallow import Schema, fields, validate


class CreateMemoSchema(Schema):
    title = fields.String(
        required=True,
        validate=validate.Length(min=1, max=200),
        metadata={"description": "메모 제목", "example": "오늘 한 일"},
    )
    content = fields.String(
        load_default="",
        metadata={"description": "본문 (선택)", "example": "10시 미팅, 12시 점심"},
    )
```

→ Swagger UI 의 schema 탭에 `description` / `example` 노출.

---

## 8. 운영 — Swagger 비활성

운영에서 API 구조 공개는 위험. **prod profile 에서 endpoint 비활성**:

```python
# app/__init__.py
def create_app(config_class=Config) -> Flask:
    app = Flask(__name__)
    # ...
    if not app.config.get("ENABLE_SWAGGER", True):
        # api.init_app 호출 안 함 → Swagger UI 등록 X
        pass
    else:
        api.init_app(app)
```

```python
# app/config.py
class ProductionConfig(Config):
    ENABLE_SWAGGER = False
```

또는 admin 만:

```python
@app.before_request
def restrict_swagger():
    if request.path.startswith("/api/docs"):
        # admin auth 검증
        pass
```

---

## 9. 자주 막히는 함정

### 함정 1 — `OPENAPI_VERSION` 누락
flask-smorest 가 spec 못 만듦. **3.0.3 명시**.

### 함정 2 — JWT Bearer 미설정
`Authorize` 버튼 없음. **`API_SPEC_OPTIONS` 의 `securitySchemes`**.

### 함정 3 — smorest 와 일반 Flask Blueprint 혼용
`Blueprint("auth", __name__)` (flask) vs `Blueprint("auth", "auth", ...)` (smorest). **두 번째 인자 (`import_name`)** 가 smorest 만 받음.

### 함정 4 — `@memos_bp.arguments(Schema)` 함수 인자 순서
`def post(self, payload, memo_id: int)` — **arguments 가 먼저, path variable 이 나중**. 거꾸로면 `TypeError`.

### 함정 5 — `MethodView` 의 `self` 누락
class-based view 라 `self` 필수.

### 함정 6 — Swagger UI 가 401
JWT loader 가 swagger path 도 막음. `/api/docs/**` 는 `@jwt_required()` 없는 endpoint.

### 함정 7 — docstring 의 첫 줄이 summary
flask-smorest 가 summary 로 사용. 짧고 명확하게.

### 함정 8 — 운영 노출
**`ENABLE_SWAGGER=False`** 또는 admin only.

---

## 10. 다음

✅ Swagger UI 통합 끝.

→ [[07-controller-patterns]] — request / response / status / Blueprint 패턴
