---
title: "10. DevTools — debug mode + Werkzeug debugger + hot reload"
kind: guide
status: current
created_at: 2026-05-27T14:05:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-flask, guidelines, devtools, hot-reload, debug]
---

# 10. DevTools — debug mode + Werkzeug debugger + hot reload

**[[guidelines|↑ Hub]]** ← [[09-logging-mdc]]

> Flask 의 dev mode + hot reload + Werkzeug debugger.

---

## 1. Debug mode — `flask --debug`

```bash
# 방법 1 — CLI
flask --app wsgi run --debug

# 방법 2 — 환경변수
export FLASK_DEBUG=1
flask --app wsgi run

# 방법 3 — Python
app.run(debug=True)
```

debug mode 시:
- ✅ Code 변경 → 자동 재시작 (Werkzeug reloader)
- ✅ Browser 에 interactive 디버거 (예외 발생 시)
- ✅ verbose error page
- ⚠️ **운영 절대 X** — RCE 가능

---

## 2. Hot reload — Werkzeug reloader

debug mode = 자동 reloader 활성. file watcher 가 source 변경 감지.

### 2.1 watch 옵션

```bash
flask run --debug --reload --extra-files config.py:templates/
# --reload: reloader 활성 (debug 와 동등)
# --extra-files: 추가 watch
```

### 2.2 watch 제외

```python
# wsgi.py
app.run(debug=True, use_reloader=True, exclude_patterns=["*.log", "__pycache__/*"])
```

---

## 3. Werkzeug debugger — browser 에서 코드 실행

예외 발생 시 browser 에 stack trace + interactive console:

```python
@app.get("/error")
def error():
    1 / 0                               # 의도적 예외
```

→ browser 에 stack trace + 각 frame 옆 콘솔 아이콘. 클릭하면 그 frame 의 context 에서 Python 코드 실행.

> **위험성**: 누구나 코드 실행 가능 → **운영 절대 비활성**. dev 에서도 `WERKZEUG_DEBUG_PIN` 확인 필요.

### 3.1 PIN

debug mode 시 콘솔에 출력:

```
* Debugger PIN: 123-456-789
```

→ debugger console 열 때 입력.

---

## 4. python-dotenv — `.env` 자동 로드

```bash
pip install python-dotenv
```

```bash
# .flaskenv (commit OK — secret 없는 dev 설정)
FLASK_APP=wsgi.py
FLASK_DEBUG=1

# .env (gitignore — secret 포함)
DATABASE_URL=postgresql+psycopg://memo:memo@localhost:5432/memo
SECRET_KEY=dev-secret
```

`flask run` 자동으로 `.flaskenv` + `.env` 읽음.

---

## 5. IPython / ipdb — interactive debugger

```bash
pip install ipython ipdb
```

```python
# breakpoint
import ipdb; ipdb.set_trace()
# 또는 Python 3.7+
breakpoint()                            # PYTHONBREAKPOINT=ipdb.set_trace 로 ipdb 사용 가능
```

→ 실행 시 해당 라인에서 멈추고 ipython 같은 인터랙티브 콘솔. `n` (next) / `s` (step) / `c` (continue) / `p var` (print).

### 5.1 VS Code debugger

`.vscode/launch.json`:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Flask",
            "type": "python",
            "request": "launch",
            "module": "flask",
            "env": { "FLASK_APP": "wsgi.py", "FLASK_DEBUG": "1" },
            "args": ["run", "--no-debugger", "--no-reload"],
            "jinja": true,
            "justMyCode": false
        }
    ]
}
```

→ VS Code 의 breakpoint 사용. `--no-debugger --no-reload` 가 핵심 (VS Code 가 따로 처리).

---

## 6. Live reload — browser 자동 새로고침

template 작업 시 유용. **Flask 기본 X** — extension 필요:

```bash
pip install flask-livereload
```

또는 단순 livereload server:

```bash
pip install livereload
```

```python
# manage.py
from livereload import Server
from app import create_app

if __name__ == "__main__":
    app = create_app()
    server = Server(app.wsgi_app)
    server.watch("app/templates/")
    server.serve(port=5000, debug=True)
```

→ template / static 변경 시 browser 자동 새로고침. API only 면 의미 적음.

---

## 7. flask shell — interactive REPL

```bash
flask --app wsgi shell
>>> from app.extensions import db
>>> from app.domain.user import User
>>> User.query.all()
[<User id=1 email='yuchan@example.com'>]
>>> User.query.filter_by(email="yuchan@example.com").first()
```

→ app context 자동 활성. 빠른 DB / service 테스트.

### 7.1 shell context 설정

```python
# app/__init__.py
@app.shell_context_processor
def shell_context():
    return {
        "db": db,
        "User": User,
        "Memo": Memo,
    }
```

→ `flask shell` 시 자동 import. `from app...` 매번 안 써도 됨.

---

## 8. pytest — TDD / 빠른 테스트

```bash
pip install pytest pytest-flask
```

```python
# tests/conftest.py
import pytest
from app import create_app
from app.config import TestConfig
from app.extensions import db


@pytest.fixture
def app():
    app = create_app(TestConfig)
    with app.app_context():
        db.create_all()
        yield app
        db.session.remove()
        db.drop_all()


@pytest.fixture
def client(app):
    return app.test_client()


@pytest.fixture
def auth_headers(client):
    client.post("/api/v1/auth/signup", json={
        "email": "t@e.com", "password": "P@ssw0rd1!", "name": "t",
    })
    res = client.post("/api/v1/auth/login", json={
        "email": "t@e.com", "password": "P@ssw0rd1!",
    })
    token = res.get_json()["data"]["accessToken"]
    return {"Authorization": f"Bearer {token}"}
```

```python
# tests/test_memos.py
def test_create_memo(client, auth_headers):
    res = client.post("/api/v1/memos",
                     json={"title": "t1", "content": "c1"},
                     headers=auth_headers)
    assert res.status_code == 201
```

```bash
pytest -v
# 또는 watch mode
pytest-watch -- -v
```

---

## 9. 코드 품질 — ruff / black / mypy

```bash
pip install ruff black mypy
```

```bash
# ruff — 빠른 linter (flake8 + isort + pyupgrade)
ruff check .
ruff check . --fix

# black — formatter
black .

# mypy — type check
mypy app/
```

`pyproject.toml`:

```toml
[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP"]

[tool.black]
line-length = 100
target-version = ["py312"]

[tool.mypy]
python_version = "3.12"
strict = true
```

---

## 10. pre-commit hook

```bash
pip install pre-commit
```

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.10.0
    hooks:
      - id: mypy
```

```bash
pre-commit install
# git commit 마다 자동 실행
```

---

## 11. 자주 막히는 함정

### 함정 1 — `debug=True` 운영
RCE 가능. **운영 절대 False**.

### 함정 2 — Werkzeug PIN 노출
`Debugger PIN` 콘솔 출력 — log 가 외부 노출되면 위험.

### 함정 3 — `app.run(debug=True)` 운영 사용
single thread / dev only. **gunicorn**.

### 함정 4 — hot reload 가 일부 변경 못 잡음
template / static 변경. **`--extra-files`** 또는 `extra_dirs`.

### 함정 5 — VS Code launcher 가 reloader 와 충돌
`--no-debugger --no-reload` 필수 (VS Code 가 별도 처리).

### 함정 6 — `.flaskenv` 가 git commit
secret 포함 시 노출. `.env` 와 분리.

### 함정 7 — pytest 의 `test_client()` 가 `app_context` 자동 안 만듦
fixture 안에서 `with app.app_context():` 명시 또는 conftest 에서 보장.

### 함정 8 — `flask shell` 에서 SQLAlchemy 변경이 transaction 안 함
`db.session.commit()` 직접 호출.

---

## 12. 다음 학습 로드맵 — 가이드 끝

✅ 10 단계 입문 + 기본 세팅 완주!

```
01. 설치 + Hello Flask           ← Python / venv / Flask
02. 프로젝트 구조 + .env          ← app factory / Blueprint / Config
03. SQLAlchemy + Migrate         ← Mapped / Flask-Migrate
04. Controller + Validation      ← marshmallow / envelope / 예외
05. Security + 배포              ← JWT / Argon2 / gunicorn / Docker
06. Swagger / OpenAPI            ← flask-smorest
07. Controller 패턴              ← request / response / status
08. Monitoring                   ← healthz / Prometheus
09. Logging + Request ID         ← dictConfig / JSON log
10. DevTools                     ← debug / hot reload  ← 지금
```

다음:

| 단계 | 학습 | 노트 |
| --- | --- | --- |
| 11 | 응답 envelope 표준 | [[../common/response-envelope]] |
| 12 | Security 표준 | [[../common/security-config]] |
| 13 | 실전 레시피 | [[../api-design/api-design]] |
| 14 | SQLAlchemy 깊이 / N+1 | [[../database/jpa]] |
| 15 | raw SQL / coexist | [[../database/mybatis]] / [[../database/jpa-mybatis-coexist]] |

---

## 13. 참고

- [[guidelines|↑ Guidelines hub]]
- [[../python-flask|↑ Flask area]]
- Flask 공식: <https://flask.palletsprojects.com/>
- Werkzeug: <https://werkzeug.palletsprojects.com/>
- pytest: <https://docs.pytest.org/>
- ruff: <https://docs.astral.sh/ruff/>
