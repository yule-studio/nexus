---
title: "06. Swagger / OpenAPI — 자동 + customize"
kind: guide
status: current
created_at: 2026-05-27T15:45:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, guidelines, swagger, openapi]
---

# 06. Swagger / OpenAPI — 자동 + customize

**[[guidelines|↑ Hub]]** ← [[05-security-and-deployment]] · → 다음 [[07-controller-patterns]]

> FastAPI = **자동 OpenAPI / Swagger UI**. 별도 라이브러리 X. 그러나 customize / Authorize 통합은 필요.

---

## 1. 자동 endpoint — 별도 설정 X

```bash
fastapi dev app/main.py
```

- **Swagger UI**: <http://localhost:8000/docs>
- **ReDoc**: <http://localhost:8000/redoc>
- **OpenAPI JSON**: <http://localhost:8000/openapi.json>

→ 모든 `@router.X` endpoint + pydantic schema 자동 노출.

---

## 2. FastAPI 인스턴스 metadata

```python
# app/main.py
app = FastAPI(
    title="Memo API",
    description="간단한 메모 CRUD + 회원가입 / JWT 인증",
    version="1.0.0",
    contact={
        "name": "yuchan",
        "email": "yuchan@example.com",
        "url": "https://github.com/yuchan",
    },
    license_info={
        "name": "Apache 2.0",
        "url": "https://www.apache.org/licenses/LICENSE-2.0",
    },
    terms_of_service="https://example.com/terms",
    openapi_tags=[
        {"name": "auth", "description": "회원가입 / 로그인 / 토큰"},
        {"name": "memos", "description": "메모 CRUD"},
        {"name": "health", "description": "헬스체크"},
    ],
)
```

→ Swagger UI 상단 + 각 tag 별 설명.

---

## 3. APIRouter — tags / response

```python
# app/api/memos.py
from fastapi import APIRouter, status

router = APIRouter()


@router.get(
    "",
    response_model=ApiResponse[MemoListResponse],
    summary="내 메모 목록",                                         # ← short summary
    description="현재 로그인된 사용자의 메모를 페이지 단위 조회. 최신순.",
    response_description="조회 결과 목록",                            # ← 200 응답 설명
    responses={                                                      # ← 추가 status code 문서
        401: {"description": "인증 필요"},
        500: {"description": "서버 오류"},
    },
)
async def list_memos(...):
    ...


@router.post(
    "",
    response_model=ApiResponse[MemoResponse],
    status_code=status.HTTP_201_CREATED,
    summary="메모 생성",
)
async def create_memo(...):
    ...
```

> **`summary`** = endpoint 제목 (Swagger UI 의 행).
> **`description`** = 상세 (다중 줄 markdown).
> **`response_description`** = 응답 설명.

### 3.1 docstring → description

```python
@router.get("/memos")
async def list_memos():
    """
    내 메모 목록을 페이지 단위 조회.

    - `limit`: 한 페이지 항목 수 (1~100)
    - `offset`: 시작 위치

    최신순으로 정렬됨.
    """
    ...
```

→ Swagger UI 의 description 에 docstring 자동.

---

## 4. JWT Bearer — Swagger Authorize 버튼

`OAuth2PasswordBearer` 가 `tokenUrl` 설정만 하면 Swagger UI 의 `Authorize` 버튼 자동 생성:

```python
# app/deps.py
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/login", auto_error=False)
```

→ Swagger UI 우측 상단 `Authorize` 클릭 → 토큰 입력 → 모든 endpoint 호출 시 자동 헤더.

### 4.1 더 명시적 — `HTTPBearer`

```python
from fastapi.security import HTTPBearer

security = HTTPBearer()


@router.get("/secret")
async def secret(credentials: Annotated[HTTPAuthorizationCredentials, Depends(security)]):
    token = credentials.credentials
    ...
```

→ Authorize 버튼이 `Bearer <token>` 명시적으로 표시.

---

## 5. Tag 별 grouping

```python
# main.py
app.include_router(auth.router, prefix="/api/v1/auth", tags=["auth"])
app.include_router(memos.router, prefix="/api/v1/memos", tags=["memos"])
app.include_router(admin.router, prefix="/api/v1/admin", tags=["admin"])
```

→ Swagger UI 의 endpoint 가 tag 별로 묶임. `openapi_tags` 의 순서대로.

---

## 6. pydantic Schema 의 OpenAPI 메타

```python
from pydantic import BaseModel, Field, ConfigDict


class CreateMemoRequest(BaseModel):
    model_config = ConfigDict(
        json_schema_extra={
            "example": {
                "title": "오늘 한 일",
                "content": "10시 미팅, 12시 점심",
            },
        },
    )

    title: str = Field(
        min_length=1, max_length=200,
        description="메모 제목",
        examples=["오늘 한 일", "TODO"],
    )
    content: str = Field(
        default="", max_length=10_000,
        description="본문 (선택)",
    )
```

→ Swagger UI 의 schema 탭 + `Example Value` 자동.

---

## 7. OpenAPI customize — 직접 override

```python
# app/main.py
from fastapi.openapi.utils import get_openapi


def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema

    openapi_schema = get_openapi(
        title=app.title,
        version=app.version,
        description=app.description,
        routes=app.routes,
    )

    # 모든 endpoint 에 Bearer auth 요구
    openapi_schema["components"]["securitySchemes"] = {
        "BearerAuth": {
            "type": "http",
            "scheme": "bearer",
            "bearerFormat": "JWT",
        }
    }
    openapi_schema["security"] = [{"BearerAuth": []}]

    app.openapi_schema = openapi_schema
    return openapi_schema


app.openapi = custom_openapi
```

---

## 8. 운영 — Swagger 비활성

운영 노출 = API 구조 + 공격 surface 공개. **prod 에선 비활성**:

```python
def create_app() -> FastAPI:
    app = FastAPI(
        title="Memo API",
        version="1.0.0",
        # 운영 환경에서 None 으로
        docs_url="/docs" if settings.ENABLE_DOCS else None,
        redoc_url="/redoc" if settings.ENABLE_DOCS else None,
        openapi_url="/openapi.json" if settings.ENABLE_DOCS else None,
        lifespan=lifespan,
    )
    return app
```

```python
# app/config.py
class Settings(BaseSettings):
    ENABLE_DOCS: bool = True
```

```bash
# 운영
export ENABLE_DOCS=false
```

또는 admin 만:

```python
@app.middleware("http")
async def restrict_docs(request: Request, call_next):
    if request.url.path in ("/docs", "/redoc", "/openapi.json"):
        # admin auth check
        if not is_admin(request):
            return JSONResponse(status_code=403, content={"detail": "Forbidden"})
    return await call_next(request)
```

---

## 9. Postman / TypeScript SDK 생성

OpenAPI JSON 다운로드:

```bash
curl http://localhost:8000/openapi.json > openapi.json
```

- **Postman**: Import → openapi.json → Collection 자동.
- **TypeScript SDK**: `openapi-typescript` 또는 `openapi-generator-cli`.
  ```bash
  npx openapi-typescript ./openapi.json -o ./src/api.d.ts
  ```
- **Java client**:
  ```bash
  openapi-generator-cli generate -i openapi.json -g java -o ./client
  ```

---

## 10. 자주 막히는 함정

### 함정 1 — `response_model` 누락
`data` 가 `Any` 로 schema → 부정확. **항상 명시**.

### 함정 2 — `tags` 가 다른 router 에 중복
같은 tag = Swagger UI 에서 한 group. 의도면 OK.

### 함정 3 — Authorize 버튼이 없음
`OAuth2PasswordBearer(tokenUrl=...)` 또는 `HTTPBearer()` Depends. 없으면 Swagger UI 의 Authorize 안 보임.

### 함정 4 — `model_config = ConfigDict(json_schema_extra=...)` 위치
class 단 (instance 단 X). pydantic v2 권장.

### 함정 5 — 한글 description 깨짐
`uvicorn --loop uvloop` 환경에서 가끔. encoding 명시.

### 함정 6 — `docs_url=None` 면 `/docs/oauth2-redirect` 도 없음
Swagger UI 의 OAuth2 callback 자동 X. 보통 무관.

### 함정 7 — 운영 노출
구조 / endpoint 모두 공개. **prod 비활성 또는 admin only**.

### 함정 8 — `openapi.json` 의 schema 가 거대
큰 프로젝트 (100+ endpoint) 에서 1MB+. **caching** 또는 `openapi_schema` property 직접.

---

## 11. 다음

✅ Swagger / OpenAPI 끝.

→ [[07-controller-patterns]] — Path / Query / Body / Depends / response_model 패턴
