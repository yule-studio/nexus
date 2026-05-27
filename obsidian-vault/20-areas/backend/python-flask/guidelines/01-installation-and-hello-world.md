---
title: "01. 설치 + venv + Hello Flask"
kind: guide
status: current
created_at: 2026-05-27T12:10:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-flask, guidelines, installation, hello-world]
---

# 01. 설치 + venv + Hello Flask

**[[guidelines|↑ Hub]]** → 다음 [[02-project-structure]]

> Python 3.12 + venv + Flask 3 설치 → 첫 endpoint.

---

## 1. Python 3.12 설치

### 1.1 macOS

```bash
brew install python@3.12
python3.12 --version            # Python 3.12.x
```

### 1.2 Ubuntu / Debian

```bash
sudo apt update
sudo apt install -y python3.12 python3.12-venv python3-pip
python3.12 --version
```

### 1.3 Windows

[python.org](https://www.python.org/downloads/) installer → `Add Python to PATH` 체크.

### 1.4 pyenv — Python 버전 관리 (권장)

여러 Python 버전 자유 전환:

```bash
# macOS
brew install pyenv

# Ubuntu
curl https://pyenv.run | bash

# 사용
pyenv install 3.12.4
pyenv global 3.12.4
python --version
```

---

## 2. venv — 프로젝트별 패키지 격리

**venv** = virtual environment. 프로젝트마다 별도 Python 환경. Java 의 Maven local repo / JS 의 `node_modules` 동등.

> **절대 system Python 에 pip install 하지 말 것** — OS 의 다른 도구가 깨질 수 있음.

```bash
mkdir flask-memo-api && cd flask-memo-api

python3.12 -m venv .venv

# 활성화
source .venv/bin/activate          # macOS / Linux
# .venv\Scripts\activate           # Windows

# 활성화 확인 — prompt 에 (.venv)
which python                        # .venv/bin/python
python --version

# 비활성화
deactivate
```

> **함정**: 매 세션 (새 터미널) 마다 `source .venv/bin/activate` 필요. IntelliJ / VS Code 는 자동.

---

## 3. Flask 설치 + Hello World

### 3.1 의존성

```bash
pip install --upgrade pip
pip install flask
pip freeze > requirements.txt
```

> `pip freeze` 가 정확한 버전을 lock. 운영 재현성.

### 3.2 hello.py

```python
# hello.py
from flask import Flask

app = Flask(__name__)


@app.get("/")
def index():
    return {"message": "Hello, Flask!"}


@app.get("/hello/<name>")
def hello_name(name):
    return {"message": f"Hello, {name}!"}


if __name__ == "__main__":
    app.run(host="127.0.0.1", port=5000, debug=True)
```

### 3.3 실행

```bash
python hello.py
# * Serving Flask app 'hello'
# * Debug mode: on
# * Running on http://127.0.0.1:5000

# 다른 터미널
curl http://127.0.0.1:5000/
# {"message":"Hello, Flask!"}

curl http://127.0.0.1:5000/hello/yuchan
# {"message":"Hello, yuchan!"}
```

### 3.4 또는 `flask` CLI

```bash
export FLASK_APP=hello
flask run --debug
# 또는
flask --app hello run --debug
```

---

## 4. 핵심 개념 — Route / View function / Decorator

```python
@app.get("/path")              # 데코레이터 — route 등록
def view_function():            # view function — endpoint handler
    return {"key": "value"}     # dict 반환 → Flask 자동 JSON
```

### 4.1 method 별 단축

| 단축 | 동등 |
| --- | --- |
| `@app.get("/x")` | `@app.route("/x", methods=["GET"])` |
| `@app.post("/x")` | `@app.route("/x", methods=["POST"])` |
| `@app.put("/x")` | `@app.route("/x", methods=["PUT"])` |
| `@app.patch("/x")` | `@app.route("/x", methods=["PATCH"])` |
| `@app.delete("/x")` | `@app.route("/x", methods=["DELETE"])` |

### 4.2 URL parameter

```python
@app.get("/users/<int:id>")
def get_user(id):                # int 자동 변환. 변환 실패 시 404.
    return {"id": id}


@app.get("/files/<path:filename>")
def get_file(filename):          # / 포함 가능
    return {"file": filename}


@app.get("/category/<string:slug>")
def category(slug):              # default — string
    return {"slug": slug}
```

| converter | 예시 |
| --- | --- |
| `string` (default) | `/foo` |
| `int` | `/42` |
| `float` | `/3.14` |
| `path` | `/a/b/c` (/ 포함) |
| `uuid` | `/abc-123-...` |

### 4.3 query string

```python
from flask import request

@app.get("/search")
def search():
    keyword = request.args.get("q")              # ?q=...
    limit = int(request.args.get("limit", 20))   # ?limit=
    return {"keyword": keyword, "limit": limit}
```

### 4.4 JSON body

```python
@app.post("/items")
def create_item():
    payload = request.get_json()                  # JSON body → dict
    if not payload:
        return {"error": "no body"}, 400
    return {"received": payload}, 201
```

---

## 5. 응답 형식

```python
# 1. dict → JSON (auto)
@app.get("/a")
def a():
    return {"key": "value"}                       # 200 + JSON

# 2. (body, status)
@app.get("/b")
def b():
    return {"key": "value"}, 201                   # 201 + JSON

# 3. (body, status, headers)
@app.get("/c")
def c():
    return {"key": "value"}, 200, {"X-Custom": "v"}

# 4. flask.Response
from flask import Response, jsonify

@app.get("/d")
def d():
    return jsonify({"key": "value"})              # 동일 — jsonify 명시

# 5. file / streaming
from flask import send_file

@app.get("/download")
def download():
    return send_file("path/to/file.pdf", as_attachment=True)
```

---

## 6. 자주 막히는 함정

### 함정 1 — system Python 에 pip install
OS 깨질 수 있음. **반드시 venv** 안에서.

### 함정 2 — venv 활성화 안 함
`pip install` 이 system 에 — `which python` 으로 확인.

### 함정 3 — `request.get_json()` 가 None
`Content-Type: application/json` 헤더 누락 시. **클라가 헤더 보내는지 확인** 또는 `force=True`.

### 함정 4 — `debug=True` 운영 노출
콘솔에 PIN 출력 → 누구나 Python 코드 실행. **운영 절대 False**.

### 함정 5 — `app.run()` 운영 사용
single-thread / dev only. **gunicorn / uwsgi 사용** ([[05-security-and-deployment]]).

### 함정 6 — URL `/x` vs `/x/`
Flask default 가 `/x` 호출 시 `/x/` 로 redirect (또는 404 — `strict_slashes` 옵션). 본 가이드는 끝 `/` 없는 형식 권장.

### 함정 7 — `pip freeze > requirements.txt` 가 모든 dev 의존성 포함
`pip-tools` 의 `pip-compile` 또는 `requirements-dev.txt` 분리 권장.

### 함정 8 — Python 3.12 미만에서 `from __future__ import annotations` 누락
3.10+ 에서도 forward reference 안전을 위해 권장.

---

## 7. 다음

✅ 설치 + Hello World 끝.

→ [[02-project-structure]] — 본격 프로젝트 패키지 구조
