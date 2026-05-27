---
title: "07. Flask Controller 패턴 — route / request / response"
kind: guide
status: current
created_at: 2026-05-27T13:30:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-flask, guidelines, controller, patterns]
---

# 07. Flask Controller 패턴 — route / request / response

**[[guidelines|↑ Hub]]** ← [[06-swagger-openapi]] · → 다음 [[08-monitoring]]

> Blueprint route / request / response / status code 패턴 정리.

---

## 1. URL converter — `<int:id>` / `<string:slug>` / ...

```python
@bp.get("/users/<int:id>")
def get_user(id: int):              # int 자동 변환. 실패 = 404.
    return ...

@bp.get("/category/<string:slug>")
def category(slug: str):             # default — string
    return ...

@bp.get("/files/<path:path>")
def file(path: str):                # / 포함 가능
    return ...

@bp.get("/<uuid:id>")
def uuid_resource(id):              # UUID format
    return ...
```

| converter | 예 |
| --- | --- |
| `string` (default) | `/foo` (slash 제외) |
| `int` | `/42` |
| `float` | `/3.14` |
| `path` | `/a/b/c` (slash 포함) |
| `uuid` | `/abc-123-...` |

---

## 2. Query string — `request.args`

```python
from flask import request

@bp.get("/search")
def search():
    keyword = request.args.get("q")                            # str | None
    limit = int(request.args.get("limit", 20))                 # default 20
    page = request.args.get("page", 0, type=int)               # type 변환 builtin
    tags = request.args.getlist("tag")                          # ?tag=a&tag=b → ["a", "b"]
    all_params = request.args.to_dict()                         # 모든 query 를 dict
    return ...
```

### 2.1 marshmallow schema 검증

[[04-controller-and-validation]] 의 `MemoListQuerySchema` 처럼:

```python
class ListQuerySchema(Schema):
    limit = fields.Integer(load_default=20, validate=validate.Range(min=1, max=100))
    offset = fields.Integer(load_default=0, validate=validate.Range(min=0))
    sort = fields.String(load_default="latest", validate=validate.OneOf(["latest", "oldest"]))


@bp.get("/items")
def list_items():
    query = ListQuerySchema().load(request.args)
    # query["limit"], query["offset"], query["sort"]
    ...
```

---

## 3. JSON body — `request.get_json()`

```python
@bp.post("/items")
def create():
    payload = request.get_json()                                # dict | None (Content-Type 안 맞으면 None)
    if not payload:
        raise BusinessException(400, "BAD_REQUEST", "Body 없음")
    ...

# 또는 force=True (Content-Type 무관)
payload = request.get_json(force=True)

# silent=True 면 parse 실패 시 None (예외 없음)
payload = request.get_json(silent=True) or {}
```

### 3.1 marshmallow load

```python
payload = CreateItemSchema().load(request.get_json())
# 실패 시 ValidationError → glob handler (422)
```

---

## 4. Form data — `request.form` (HTML form / `application/x-www-form-urlencoded`)

```python
@bp.post("/login-form")
def login_form():
    email = request.form.get("email")
    password = request.form.get("password")
    ...
```

→ REST API 면 JSON 권장. form 은 HTML template 또는 legacy 호환.

---

## 5. 파일 업로드 — `request.files` (multipart/form-data)

```python
from werkzeug.utils import secure_filename


@bp.post("/upload")
def upload():
    file = request.files.get("file")
    if file is None or file.filename == "":
        raise BusinessException(400, "NO_FILE", "파일 없음")

    filename = secure_filename(file.filename)
    file.save(f"/tmp/{filename}")
    return ok({"filename": filename})
```

```yaml
# config — file size limit
class Config:
    MAX_CONTENT_LENGTH = 50 * 1024 * 1024            # 50 MB
```

→ 큰 파일은 [[../api-design/file-upload-s3]] 의 presigned URL 권장.

---

## 6. Header / Cookie

```python
@bp.get("/secret")
def secret():
    auth = request.headers.get("Authorization")
    accept_lang = request.headers.get("Accept-Language", "ko")
    session_cookie = request.cookies.get("session")
    return ...
```

---

## 7. 응답 — return 형식 5가지

```python
# 1. dict → JSON (200)
@bp.get("/a")
def a():
    return {"key": "value"}

# 2. (body, status)
@bp.get("/b")
def b():
    return {"key": "value"}, 201

# 3. (body, status, headers)
@bp.get("/c")
def c():
    return {"key": "value"}, 200, {"X-Custom": "v"}

# 4. flask.jsonify — Response 객체
from flask import jsonify
@bp.get("/d")
def d():
    return jsonify({"key": "value"})

# 5. file / binary
from flask import send_file, send_from_directory
@bp.get("/download")
def download():
    return send_file("/path/to/file.pdf", as_attachment=True)
```

---

## 8. status code 결정 — REST 관행

| 상황 | status |
| --- | --- |
| 정상 조회 / 갱신 | **200 OK** |
| 신규 생성 | **201 Created** (+ Location 헤더) |
| 처리 시작 (async) | **202 Accepted** |
| 정상 삭제 / body 없음 | **204 No Content** |
| client 입력 잘못 | **400 Bad Request** |
| 인증 안 됨 | **401 Unauthorized** |
| 권한 없음 | **403 Forbidden** |
| 리소스 없음 | **404 Not Found** |
| method 안 됨 | **405 Method Not Allowed** |
| 충돌 | **409 Conflict** |
| validation 실패 | **422 Unprocessable Entity** |
| rate limit | **429 Too Many Requests** |
| 서버 오류 | **500 Internal Server Error** |
| 외부 의존 실패 | **502 Bad Gateway** |
| 일시 사용 불가 | **503 Service Unavailable** |

본 vault: validation = 422, parsing 자체 실패 = 400.

---

## 9. streaming 응답 — 큰 데이터

```python
from flask import Response, stream_with_context

@bp.get("/exports/users.csv")
def export_users():
    def generate():
        yield "﻿"                                                # UTF-8 BOM
        yield "id,email\n"
        for user in user_repo.stream_all():                       # generator
            yield f"{user.id},{user.email}\n"

    return Response(
        stream_with_context(generate()),
        mimetype="text/csv; charset=UTF-8",
        headers={"Content-Disposition": 'attachment; filename="users.csv"'},
    )
```

> **`stream_with_context`** — request context 유지하면서 generator. 없으면 generator 안에서 `request.x` 접근 시 RuntimeError.

---

## 10. redirect / abort

```python
from flask import redirect, abort, url_for

@bp.get("/old")
def old():
    return redirect(url_for("memos.list_memos"), code=301)         # 301 permanent

@bp.get("/protected")
def protected():
    if not authorized:
        abort(403)                                                  # werkzeug.HTTPException → handler
    return ok()
```

---

## 11. URL 생성 — `url_for`

```python
@bp.get("/items/<int:id>")
def get_item(id: int):
    return ...

# 다른 endpoint 에서 URL 생성
url = url_for("items.get_item", id=42)                              # → /api/v1/items/42

# 전체 URL (scheme + host)
url = url_for("items.get_item", id=42, _external=True)              # → https://x.com/api/v1/items/42

# query 포함
url = url_for("items.list_items", page=1, size=20)                  # → /items?page=1&size=20
```

→ hard-coded URL 보다 안전 — endpoint 이름 바뀌면 IDE / linter 가 잡음.

---

## 12. before_request / after_request — hook

```python
@app.before_request
def before():
    """모든 요청 전 — 시간 측정 시작 등."""
    g.request_start = time.monotonic()


@app.after_request
def after(response):
    """모든 응답 후 (예외 X 일 때만)."""
    dur = time.monotonic() - g.request_start
    response.headers["X-Response-Time"] = f"{dur * 1000:.0f}ms"
    return response


@app.teardown_request
def teardown(exception):
    """예외 발생해도 무조건 실행 — cleanup."""
    db.session.close()
```

### 12.1 Blueprint 단

```python
@bp.before_request
def bp_before():
    """이 BP 의 요청만."""
    pass
```

---

## 13. `g` — request scope 데이터

```python
from flask import g

@app.before_request
def attach_user():
    if has_jwt():
        g.current_user = load_user_from_jwt()


@bp.get("/me")
def me():
    return ok({"id": g.current_user.id})
```

> **`g`** 는 thread-local request scope. 다음 요청에선 사라짐. session 과 다름 (session 은 cookie 기반).

---

## 14. 자주 막히는 함정

### 함정 1 — `request.get_json()` 가 None
Content-Type 헤더 누락. `force=True` 또는 헤더 강제.

### 함정 2 — URL trailing slash
Flask default `strict_slashes=True`. `/x` → `/x/` redirect. `app.url_map.strict_slashes = False` 또는 일관.

### 함정 3 — `url_for` 의 endpoint 이름
`Blueprint.endpoint` = `<bp_name>.<function_name>`. `url_for("memos.create_memo")`.

### 함정 4 — `@bp.before_request` vs `@app.before_request`
BP 단 vs 전역. 의도 명확히.

### 함정 5 — `g` 가 thread 별로 다름
multi-thread Flask (default) 에서 thread-local. async (Quart) 와 혼용 시 위험.

### 함정 6 — `request.args.get("limit")` 가 str
`int(...)` 필요. 또는 `request.args.get("limit", 20, type=int)`.

### 함정 7 — `request.files["file"]` 가 KeyError
없으면 `request.files.get("file")` 또는 try/except.

### 함정 8 — streaming generator 안 `db.session` 사용
generator 가 응답 시점에 평가 → request context 끝난 후. `stream_with_context` 필수.

### 함정 9 — `redirect(url_for(...))` 의 URL 변경 시 cache 깨짐
301 Permanent 는 browser cache. **302 / 307 권장** (재요청 보장).

### 함정 10 — `secure_filename` 누락
사용자 파일명 그대로 사용 = path traversal. **항상 `secure_filename`**.

---

## 15. 다음

✅ Controller 패턴 끝.

→ [[08-monitoring]] — prometheus-flask-exporter + healthz
