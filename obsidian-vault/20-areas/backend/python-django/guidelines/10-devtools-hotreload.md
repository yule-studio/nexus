---
title: "10. DevTools — runserver + pytest-django + debug-toolbar + ruff"
kind: guide
status: current
created_at: 2026-05-26T13:00:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-django, guidelines, devtools, runserver, pytest, debug-toolbar]
---

# 10. DevTools — runserver + pytest-django + debug-toolbar + ruff

**[[guidelines|↑ Hub]]** ← [[09-logging-mdc]]

> dev cycle 가속 + 테스트 + 코드 품질 도구.

---

## 1. runserver — Django dev server

```bash
python manage.py runserver               # 127.0.0.1:8000
python manage.py runserver 0.0.0.0:8000  # bind
python manage.py runserver --noreload    # auto-reload 비활성
```

- 코드 변경 시 자동 reload (StatReloader)
- static / media 자동 serve
- **dev only** — gunicorn 등이 운영

---

## 2. shell — Django interactive

```bash
python manage.py shell                   # Python REPL + django setup
python manage.py shell -i ipython        # IPython
python manage.py shell -i bpython
```

`pip install ipython` → 더 좋은 REPL.

`shell_plus` (django-extensions) — 더 강력:
```bash
pip install django-extensions
```

```python
# settings/dev.py
INSTALLED_APPS += ["django_extensions"]
```

```bash
python manage.py shell_plus              # 모든 모델 자동 import
```

→ 매번 `from apps.memos.models import Memo` 할 필요 X.

---

## 3. debug-toolbar — SQL / cache / template 시각화

```bash
pip install django-debug-toolbar
```

```python
# settings/dev.py
INSTALLED_APPS += ["debug_toolbar"]
MIDDLEWARE += ["debug_toolbar.middleware.DebugToolbarMiddleware"]
INTERNAL_IPS = ["127.0.0.1"]

DEBUG_TOOLBAR_CONFIG = {
    "SHOW_TOOLBAR_CALLBACK": lambda r: settings.DEBUG and r.META.get("REMOTE_ADDR") in INTERNAL_IPS,
}
```

```python
# config/urls.py
if settings.DEBUG:
    import debug_toolbar
    urlpatterns += [path("__debug__/", include(debug_toolbar.urls))]
```

→ 모든 페이지 우측에 toolbar (SQL / Cache / Template / Profile).

> DRF 의 API endpoint 는 HTML 응답이 아니라 안 보임. **DRF Browsable API** (`?format=api` 또는 직접 접속) 또는 별도 panel.

---

## 4. pytest-django — 본 가이드의 표준 test 도구

```bash
pip install pytest pytest-django pytest-asyncio
```

`pyproject.toml`:
```toml
[tool.pytest.ini_options]
DJANGO_SETTINGS_MODULE = "config.settings.test"
python_files = "test_*.py tests.py"
addopts = "-v --tb=short --reuse-db"          # --reuse-db = 매번 DB 재생성 X
```

> `pytest-django` 가 `unittest.TestCase` 보다 권장 — fixture / 간결.

### 4.1 fixture

```python
# tests/conftest.py
import pytest
from rest_framework.test import APIClient

from apps.accounts.models import User


@pytest.fixture
def api_client():
    return APIClient()


@pytest.fixture
def user(db):                                  # `db` = pytest-django 의 DB 활성화
    return User.objects.create_user(email="t@e.com", password="P@ssw0rd1!", name="t")


@pytest.fixture
def auth_client(api_client, user):
    api_client.force_authenticate(user=user)
    return api_client
```

### 4.2 test

```python
# tests/test_memos.py
import pytest


@pytest.mark.django_db
def test_create_memo(auth_client):
    res = auth_client.post("/api/memos/", {"title": "t1", "content": "c1"}, format="json")
    assert res.status_code == 201
    assert res.json()["data"]["title"] == "t1"


@pytest.mark.django_db
def test_unauthorized(api_client):
    res = api_client.get("/api/memos/")
    assert res.status_code == 401
```

```bash
pytest -v
pytest tests/test_memos.py::test_create_memo -v
pytest -k "memo" -v                            # 이름에 memo 포함
pytest --reuse-db                              # DB 재사용 (빠름)
pytest --create-db                             # 강제 재생성
```

---

## 5. factory_boy — fixture 의 표준

```bash
pip install factory_boy
```

```python
# tests/factories.py
import factory
from apps.accounts.models import User
from apps.memos.models import Memo


class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = User

    email = factory.Sequence(lambda n: f"u{n}@e.com")
    name = factory.Faker("name")
    password = factory.PostGenerationMethodCall("set_password", "P@ssw0rd1!")


class MemoFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Memo

    user = factory.SubFactory(UserFactory)
    title = factory.Faker("sentence", nb_words=4)
    content = factory.Faker("paragraph")
```

```python
# 사용
@pytest.mark.django_db
def test_list_memos(auth_client, user):
    MemoFactory.create_batch(3, user=user)
    MemoFactory.create_batch(2)                # 다른 사용자

    res = auth_client.get("/api/memos/")
    assert res.json()["count"] == 3            # 본인 것만
```

---

## 6. coverage

```bash
pip install coverage
coverage run -m pytest
coverage report -m
coverage html                                  # htmlcov/index.html
```

`.coveragerc`:
```ini
[run]
source = apps
omit =
    */migrations/*
    */tests/*

[report]
exclude_lines =
    pragma: no cover
    if TYPE_CHECKING:
    raise NotImplementedError
```

---

## 7. 코드 품질 — ruff / black / mypy

```bash
pip install ruff black mypy django-stubs djangorestframework-stubs
```

```bash
ruff check . --fix
black .
mypy apps/
```

`pyproject.toml`:
```toml
[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "B", "DJ"]            # DJ = Django specific
ignore = ["E501"]                              # line length is black

[tool.black]
line-length = 100
target-version = ["py312"]

[tool.mypy]
plugins = ["mypy_django_plugin.main", "mypy_drf_plugin.main"]
strict = false
ignore_missing_imports = true

[tool.django-stubs]
django_settings_module = "config.settings.dev"
```

---

## 8. pre-commit

```bash
pip install pre-commit
```

`.pre-commit-config.yaml`:
```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.4
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  - repo: https://github.com/adamchainz/django-upgrade
    rev: 1.18.0
    hooks:
      - id: django-upgrade
        args: [--target-version, "5.0"]
```

```bash
pre-commit install
```

---

## 9. IDE 통합 — VS Code

`.vscode/launch.json`:
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Django runserver",
            "type": "debugpy",
            "request": "launch",
            "program": "${workspaceFolder}/manage.py",
            "args": ["runserver", "0.0.0.0:8000", "--noreload"],
            "django": true,
            "justMyCode": false
        },
        {
            "name": "Pytest",
            "type": "debugpy",
            "request": "launch",
            "module": "pytest",
            "args": ["-v"],
            "django": true,
            "justMyCode": false
        }
    ]
}
```

→ `--noreload` 권장 (debugger 와 reload 충돌).

---

## 10. management command — 직접 작성

```bash
mkdir -p apps/memos/management/commands
touch apps/memos/management/__init__.py
touch apps/memos/management/commands/__init__.py
```

```python
# apps/memos/management/commands/cleanup_old_memos.py
from django.core.management.base import BaseCommand
from django.utils import timezone
from datetime import timedelta

from apps.memos.models import Memo


class Command(BaseCommand):
    help = "오래된 메모 삭제"

    def add_arguments(self, parser):
        parser.add_argument("--days", type=int, default=90)
        parser.add_argument("--dry-run", action="store_true")

    def handle(self, *args, days, dry_run, **opts):
        cutoff = timezone.now() - timedelta(days=days)
        qs = Memo.objects.filter(created_at__lt=cutoff)
        self.stdout.write(f"대상: {qs.count()}건 (cutoff={cutoff})")
        if not dry_run:
            qs.delete()
            self.stdout.write(self.style.SUCCESS("삭제 완료"))
```

```bash
python manage.py cleanup_old_memos --days 30 --dry-run
python manage.py cleanup_old_memos --days 30
```

→ cron / k8s CronJob 으로 schedule.

---

## 11. dbshell / inspectdb

```bash
python manage.py dbshell                       # DB 의 CLI (mysql / psql)
python manage.py inspectdb > legacy_models.py  # 기존 DB → model 자동 생성 (legacy 마이그레이션)
```

---

## 12. HTTP Client — VS Code `.http`

`requests.http`:
```http
### 1. signup
POST http://localhost:8000/api/auth/signup/
Content-Type: application/json

{"email":"yuchan@example.com","password":"P@ssw0rd1!","name":"yuchan"}

### 2. login
POST http://localhost:8000/api/auth/login/
Content-Type: application/json

{"email":"yuchan@example.com","password":"P@ssw0rd1!"}

> {% client.global.set("token", response.body.data.accessToken); %}

### 3. 메모 생성
POST http://localhost:8000/api/memos/
Authorization: Bearer {{token}}
Content-Type: application/json

{"title":"첫 메모","content":"안녕"}
```

→ git commit 가능. Postman 대체.

---

## 13. 자주 막히는 함정

### 함정 1 — `runserver --noreload` 안 쓰고 debugger
breakpoint 가 두 process 에서 hit (auto-reload subprocess). **`--noreload`** 권장.

### 함정 2 — pytest-django 의 `@pytest.mark.django_db` 누락
DB 접근 → `RuntimeError: Database access not allowed`. 마커 필수 (또는 `db` fixture 의존).

### 함정 3 — `--reuse-db` 후 schema 변경
old schema 의 DB 가 재사용 → migration 실수. **`--create-db`** 한 번.

### 함정 4 — `force_authenticate` vs 실제 login
실제 흐름 검증이면 `client.post("/api/auth/login/", ...)`. 단위 테스트면 force_authenticate 가 빠름.

### 함정 5 — `factory_boy` 의 `PostGenerationMethodCall` 와 unique
`Sequence` 가 중복되면 IntegrityError. global counter 라 test 간 격리 X — `reset_sequence`.

### 함정 6 — `mypy` + django-stubs 의 settings module
`[tool.django-stubs]` 의 `django_settings_module` 명시. 누락 시 strict 모드 다양한 false-positive.

### 함정 7 — debug-toolbar 가 운영
`DEBUG=True` 가 아니어도 middleware 가 활성화되면 노출. **`if DEBUG`** guard.

### 함정 8 — `pytest-asyncio` + Django async view
Django 4.0+ async view 는 `pytest.mark.asyncio` + `async_to_sync`. 동기 view 는 그냥.

### 함정 9 — `management/commands/__init__.py` 누락
command 인식 X (silent). 빈 파일이라도 필수.

### 함정 10 — `coverage` 가 manage.py / settings 도 측정
`omit` 으로 제외. 100% 목표 false signal.

---

## 14. 다음 학습 — 가이드 끝

✅ 10 단계 완주!

| 다음 | 노트 |
| --- | --- |
| 11 | [[../common/common]] |
| 12 | [[../api-design/api-design]] |
| 13 | [[../database/database]] |
| 14 | [[../api-docs/api-docs]] |
| 15 | [[../pitfalls/pitfalls]] |

---

## 15. 참고

- [[guidelines|↑ Guidelines hub]]
- [[../python-django|↑ Django area]]
- Django 공식: <https://docs.djangoproject.com/en/5.0/>
- DRF 공식: <https://www.django-rest-framework.org/>
- pytest-django: <https://pytest-django.readthedocs.io/>
- factory_boy: <https://factoryboy.readthedocs.io/>
- django-extensions: <https://django-extensions.readthedocs.io/>
- django-debug-toolbar: <https://django-debug-toolbar.readthedocs.io/>
