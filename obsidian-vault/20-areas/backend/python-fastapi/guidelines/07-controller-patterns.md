---
title: "07. Path / Query / Body / Depends / response_model 패턴"
kind: guide
status: current
created_at: 2026-05-27T16:00:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, guidelines, controller, patterns]
---

# 07. Path / Query / Body / Depends / response_model 패턴

**[[guidelines|↑ Hub]]** ← [[06-swagger-openapi]] · → 다음 [[08-monitoring]]

> FastAPI 의 다양한 request / response 패턴 정리.

---

## 1. Path parameter — `/users/{id}`

```python
@router.get("/users/{id}")
async def get_user(id: int):                    # type hint = int 자동 변환
    return {"id": id}


# Path() 로 메타 추가
from fastapi import Path

@router.get("/users/{id}")
async def get_user(
    id: Annotated[int, Path(ge=1, le=999_999, description="user id")],
):
    return {"id": id}
```

### 1.1 여러 path / 다른 이름

```python
@router.get("/users/{user_id}/memos/{memo_id}")
async def get_memo(user_id: int, memo_id: int):
    ...


# JSON field 이름 다를 때 — alias
@router.get("/items/{itemId}")
async def get_item(
    itemId: Annotated[int, Path(alias="itemId")],     # URL = itemId, var = item_id 도 가능
):
    ...
```

---

## 2. Query parameter — `?key=value`

```python
@router.get("/items")
async def list_items(
    q: str | None = None,                              # ?q=... — optional
    limit: int = 20,                                    # ?limit= — int 자동
    offset: int = 0,
):
    return {...}


# Query() 로 메타 / validation
from fastapi import Query

@router.get("/items")
async def list_items(
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
    offset: Annotated[int, Query(ge=0)] = 0,
    q: Annotated[str | None, Query(min_length=2, max_length=100, description="검색어")] = None,
    tags: Annotated[list[str] | None, Query()] = None,    # ?tags=a&tags=b
):
    ...
```

### 2.1 enum 자동 변환

```python
from enum import Enum


class SortBy(str, Enum):
    LATEST = "latest"
    OLDEST = "oldest"
    NAME = "name"


@router.get("/items")
async def list_items(sort: SortBy = SortBy.LATEST):
    return {"sort": sort.value}
```

→ Swagger UI 가 dropdown. 잘못된 값 = 422.

### 2.2 query 를 pydantic model 로 묶기

```python
from pydantic import BaseModel


class ListQuery(BaseModel):
    limit: int = Field(default=20, ge=1, le=100)
    offset: int = Field(default=0, ge=0)
    q: str | None = None


@router.get("/items")
async def list_items(query: Annotated[ListQuery, Depends()]):
    ...
```

→ FastAPI 가 query string → pydantic model 자동. 재사용 / type 깔끔.

---

## 3. Body — pydantic model

```python
class CreateItemRequest(BaseModel):
    name: str
    price: int = Field(ge=0)


@router.post("/items")
async def create(item: CreateItemRequest):
    return item


# 여러 body — wrapping 또는 Body()
from fastapi import Body

@router.post("/items")
async def create(
    item: CreateItemRequest,
    note: Annotated[str, Body(embed=True)] = "",       # JSON: {"item": {...}, "note": "..."}
):
    ...
```

### 3.1 Body 만 받을 때

```python
@router.post("/raw")
async def raw_body(body: Annotated[dict, Body()]):     # dict 그대로
    return body
```

---

## 4. Header / Cookie

```python
from fastapi import Header, Cookie


@router.get("/me")
async def me(
    authorization: Annotated[str | None, Header()] = None,
    user_agent: Annotated[str | None, Header()] = None,
    session_id: Annotated[str | None, Cookie()] = None,
):
    ...
```

> **자동 변환**: header `User-Agent` → param `user_agent` (Python 명명).

---

## 5. File upload

```python
from fastapi import UploadFile, File, Form


@router.post("/upload")
async def upload(
    file: UploadFile = File(...),
    description: Annotated[str, Form()] = "",
):
    contents = await file.read()                       # 메모리에 (큰 파일 X)
    return {"filename": file.filename, "size": len(contents)}


# streaming
@router.post("/upload-stream")
async def upload_stream(file: UploadFile):
    async with aiofiles.open(f"/tmp/{file.filename}", "wb") as out:
        while chunk := await file.read(8192):
            await out.write(chunk)
```

> 큰 파일은 [[../api-design/file-upload-s3]] 의 presigned URL.

---

## 6. Depends — DI

```python
SessionDep = Annotated[AsyncSession, Depends(get_db)]
CurrentUserDep = Annotated[User, Depends(get_current_user)]


@router.get("/memos")
async def list_memos(db: SessionDep, user: CurrentUserDep):
    return await memo_service.list_memos(db, user.id)
```

### 6.1 sub-dependency / dependency chain

```python
async def get_db_session():
    async with AsyncSessionLocal() as session:
        yield session


async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
    db: Annotated[AsyncSession, Depends(get_db_session)],
) -> User:
    ...


async def require_admin(user: Annotated[User, Depends(get_current_user)]) -> User:
    if user.role != "ADMIN":
        raise ForbiddenException()
    return user


@router.get("/admin/users")
async def admin_list(admin: Annotated[User, Depends(require_admin)]):
    ...
```

→ FastAPI 가 graph 자동. 같은 dependency 는 request 당 한 번만.

### 6.2 dependency cache 비활성

```python
# 항상 새로 호출
async def fresh_dependency(... = Depends(my_dep, use_cache=False)):
    ...
```

### 6.3 router 전체 dependency

```python
admin_router = APIRouter(
    prefix="/admin",
    dependencies=[Depends(require_admin)],             # 모든 endpoint 에 적용
)


@admin_router.get("/users")
async def admin_list():                                # require_admin 자동 적용
    ...
```

---

## 7. 응답 — response_model + status_code

```python
@router.get("/items/{id}", response_model=ItemResponse)
async def get_item(id: int) -> ItemResponse:
    return await item_service.find(id)


@router.post(
    "/items",
    response_model=ItemResponse,
    status_code=status.HTTP_201_CREATED,                # 201
)
async def create(payload: CreateItemRequest):
    return await item_service.create(payload)


@router.delete("/items/{id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete(id: int):
    await item_service.delete(id)
    # return 안 함 — 204 No Content
```

### 7.1 response_model 의 `exclude` / `include`

```python
@router.get(
    "/users/{id}",
    response_model=UserResponse,
    response_model_exclude={"password_hash", "internal_field"},     # 노출 X
    response_model_exclude_none=True,                                # None 필드 X
)
async def get_user(id: int):
    ...
```

### 7.2 binary / streaming / file 응답

```python
from fastapi.responses import StreamingResponse, FileResponse


@router.get("/download")
async def download():
    return FileResponse("/path/to/file.pdf", media_type="application/pdf",
                       filename="report.pdf")


@router.get("/stream")
async def stream():
    async def generate():
        for i in range(100):
            yield f"line {i}\n".encode()

    return StreamingResponse(
        generate(),
        media_type="text/plain",
        headers={"Content-Disposition": "attachment; filename=data.txt"},
    )
```

### 7.3 redirect

```python
from fastapi.responses import RedirectResponse


@router.get("/old")
async def old_redirect():
    return RedirectResponse(url="/new", status_code=301)
```

---

## 8. status code — REST 관행

| 상황 | status |
| --- | --- |
| 정상 조회 / 갱신 | 200 OK |
| 신규 생성 | 201 Created |
| async 시작 | 202 Accepted |
| 정상 삭제 | 204 No Content |
| client 잘못 | 400 Bad Request |
| 인증 안 됨 | 401 Unauthorized |
| 권한 없음 | 403 Forbidden |
| 리소스 없음 | 404 Not Found |
| 충돌 | 409 Conflict |
| validation 실패 | 422 Unprocessable Entity |
| rate limit | 429 Too Many Requests |
| 서버 오류 | 500 Internal Server Error |

본 vault: validation = 422 (FastAPI 기본).

---

## 9. BackgroundTasks — 응답 후 작업

```python
from fastapi import BackgroundTasks


def send_email_sync(to: str, subject: str):
    # blocking
    ...


async def send_email_async(to: str, subject: str):
    # non-blocking
    ...


@router.post("/notify")
async def notify(email: str, bg: BackgroundTasks):
    bg.add_task(send_email_sync, email, "Welcome")     # sync 도 OK (thread pool)
    bg.add_task(send_email_async, email, "Welcome")    # async
    return {"queued": True}
```

→ 응답 후 실행. small fire-and-forget. **재시도 / monitoring 필요면 Celery**.

---

## 10. middleware — 모든 요청에

```python
import time
from fastapi import Request


@app.middleware("http")
async def add_response_time(request: Request, call_next):
    start = time.monotonic()
    response = await call_next(request)
    dur_ms = int((time.monotonic() - start) * 1000)
    response.headers["X-Response-Time"] = f"{dur_ms}ms"
    return response
```

→ before_request / after_request 통합. 단 stream response 와 일부 호환 이슈.

---

## 11. 자주 막히는 함정

### 함정 1 — `Annotated` 없이 `dep: X = Depends(...)`
구식. v0.95+ `Annotated[X, Depends(...)]` 권장 — IDE / mypy 친화.

### 함정 2 — `Body(embed=True)` 누락
하나의 body 면 자동 unwrap. 여러 body 면 `embed=True` 로 wrap 안내.

### 함정 3 — `Depends()` 의 sync vs async
sync `def` OK (thread pool 자동). 단 안 blocking IO 면 event loop block — DB / HTTP 는 async.

### 함정 4 — `response_model` 과 return type 불일치
`response_model=X` 면 X 의 field 만 노출. return type 의 다른 field 무시. 의도면 OK.

### 함정 5 — `BackgroundTasks` 가 fail 시 silent
log / Sentry. 중요한 작업은 Celery.

### 함정 6 — middleware 안 sync DB
event loop block. async client 만.

### 함정 7 — Path 의 type 변환 실패
`/items/abc` 에 `id: int` = 422 자동. 의도된 동작.

### 함정 8 — `UploadFile.read()` 가 메모리 폭발
큰 파일 = OOM. **chunk streaming** (`while chunk := await file.read(8192)`).

### 함정 9 — `Form()` 과 `Body()` 함께 사용 시 multipart 강제
FastAPI 가 자동 multipart 처리. JSON body 와 form 데이터 같이 못 받음.

### 함정 10 — Response 객체 직접 반환 + response_model
`response_model` 무시됨. JSONResponse 등 직접 만들면 schema X.

---

## 12. 다음

✅ Controller 패턴 끝.

→ [[08-monitoring]] — prometheus-fastapi-instrumentator + healthz
