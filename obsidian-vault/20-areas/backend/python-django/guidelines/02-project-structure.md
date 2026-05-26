---
title: "02. 프로젝트 구조 — settings 분리 + app 분리 + .env"
kind: guide
status: current
created_at: 2026-05-26T10:25:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-django, guidelines, project-structure, settings]
---

# 02. 프로젝트 구조 — settings 분리 + app 분리 + .env

**[[guidelines|↑ Hub]]** ← [[01-installation-and-hello-world]] · → 다음 [[03-models-and-orm]]

---

## 1. 권장 디렉토리 — 본 가이드

```
memo-django/
├── manage.py
├── requirements/
│   ├── base.txt                 # 공통
│   ├── dev.txt                  # -r base.txt + dev 도구
│   └── prod.txt                 # -r base.txt + gunicorn 등
├── .env.example
├── .gitignore
├── Dockerfile
├── docker-compose.yml
├── config/                       # 프로젝트 settings 패키지
│   ├── __init__.py
│   ├── settings/                 # ← settings 분리
│   │   ├── __init__.py
│   │   ├── base.py
│   │   ├── dev.py
│   │   ├── prod.py
│   │   └── test.py
│   ├── urls.py                   # 최상단 urlpatterns
│   ├── asgi.py
│   └── wsgi.py
├── apps/                         # 모든 도메인 app
│   ├── __init__.py
│   ├── accounts/                 # 인증 / User
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   ├── admin.py
│   │   ├── apps.py
│   │   └── migrations/
│   └── memos/                    # 메모
│       ├── models.py
│       ├── serializers.py
│       ├── views.py
│       ├── urls.py
│       ├── admin.py
│       ├── apps.py
│       └── migrations/
└── tests/
    ├── conftest.py
    └── test_memos.py
```

> 핵심:
> - **settings 를 `settings.py` 단일 파일이 아니라 `settings/` 패키지**
> - **모든 app 을 `apps/` 안에**
> - **`requirements` 도 dev / prod 분리**

---

## 2. requirements 분리

```bash
mkdir requirements
```

`requirements/base.txt`:
```
django==5.0.6
djangorestframework==3.15.2
django-environ==0.11.2
djangorestframework-simplejwt==5.3.1
drf-spectacular==0.27.2
mysqlclient==2.2.4               # 또는 psycopg2-binary
```

`requirements/dev.txt`:
```
-r base.txt
ipython==8.24.0
django-debug-toolbar==4.4.2
ruff==0.4.4
black==24.4.2
pytest==8.2.0
pytest-django==4.8.0
```

`requirements/prod.txt`:
```
-r base.txt
gunicorn==22.0.0
sentry-sdk==2.3.1
django-prometheus==2.3.1
```

설치:
```bash
pip install -r requirements/dev.txt          # 개발
pip install -r requirements/prod.txt         # 운영
```

---

## 3. settings 분리 — 핵심

`settings.py` (단일) → `settings/{base,dev,prod,test}.py` (패키지).

### 3.1 base.py — 공통

```python
# config/settings/base.py
from pathlib import Path
import environ


BASE_DIR = Path(__file__).resolve().parent.parent.parent
env = environ.Env()
environ.Env.read_env(BASE_DIR / ".env")              # .env 자동 로딩

SECRET_KEY = env("SECRET_KEY")
DEBUG = False
ALLOWED_HOSTS: list[str] = []

INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",

    # 3rd party
    "rest_framework",
    "rest_framework_simplejwt",
    "drf_spectacular",

    # local apps
    "apps.accounts",
    "apps.memos",
]

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]

ROOT_URLCONF = "config.urls"
WSGI_APPLICATION = "config.wsgi.application"
ASGI_APPLICATION = "config.asgi.application"

TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [],
        "APP_DIRS": True,
        "OPTIONS": {
            "context_processors": [
                "django.template.context_processors.debug",
                "django.template.context_processors.request",
                "django.contrib.auth.context_processors.auth",
                "django.contrib.messages.context_processors.messages",
            ],
        },
    },
]

DATABASES = {
    "default": env.db("DATABASE_URL"),
}

AUTH_USER_MODEL = "accounts.User"                    # Custom User (03 에서 정의)

# DRF
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": (
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ),
    "DEFAULT_PERMISSION_CLASSES": (
        "rest_framework.permissions.IsAuthenticated",
    ),
    "DEFAULT_PAGINATION_CLASS":
        "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 20,
    "DEFAULT_SCHEMA_CLASS": "drf_spectacular.openapi.AutoSchema",
}

# i18n / TZ
LANGUAGE_CODE = "ko-kr"
TIME_ZONE = "Asia/Seoul"
USE_I18N = True
USE_TZ = True

STATIC_URL = "static/"
STATIC_ROOT = BASE_DIR / "staticfiles"

DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"
```

### 3.2 dev.py

```python
# config/settings/dev.py
from .base import *                                  # noqa

DEBUG = True
ALLOWED_HOSTS = ["*"]

INSTALLED_APPS += ["debug_toolbar"]
MIDDLEWARE += ["debug_toolbar.middleware.DebugToolbarMiddleware"]
INTERNAL_IPS = ["127.0.0.1"]
```

### 3.3 prod.py

```python
# config/settings/prod.py
from .base import *                                  # noqa

DEBUG = False
ALLOWED_HOSTS = env.list("ALLOWED_HOSTS")            # 콤마 구분

SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

### 3.4 test.py

```python
# config/settings/test.py
from .base import *                                  # noqa

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": ":memory:",
    }
}
PASSWORD_HASHERS = ["django.contrib.auth.hashers.MD5PasswordHasher"]    # test 만 빠르게
```

---

## 4. `DJANGO_SETTINGS_MODULE` — 어떤 settings 사용할지

`manage.py` / `wsgi.py` / `asgi.py` 모두 환경변수로 결정:

```python
# manage.py
import os
import sys


def main():
    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings.dev")
    # ...
```

```bash
# dev
python manage.py runserver
# DJANGO_SETTINGS_MODULE=config.settings.dev (default)

# prod
DJANGO_SETTINGS_MODULE=config.settings.prod gunicorn config.wsgi
```

---

## 5. .env — django-environ

`.env.example` (git commit):
```
DJANGO_SETTINGS_MODULE=config.settings.dev
SECRET_KEY=change-me-development-only
ALLOWED_HOSTS=localhost,127.0.0.1
DATABASE_URL=mysql://memo:memo@localhost:3306/memo
JWT_SIGNING_KEY=change-me-jwt-key
```

`.env` (gitignore):
```
SECRET_KEY=actual-very-long-random-key
DATABASE_URL=mysql://memo:memo@db:3306/memo
JWT_SIGNING_KEY=...
```

`.gitignore`:
```
.env
.venv/
__pycache__/
*.pyc
db.sqlite3
staticfiles/
media/
*.log
```

---

## 6. apps/ 패키지 만들기

```bash
mkdir apps
touch apps/__init__.py
cd apps
python ../manage.py startapp accounts
python ../manage.py startapp memos
cd ..
```

`apps/accounts/apps.py`:
```python
from django.apps import AppConfig


class AccountsConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "apps.accounts"                              # ← 풀 경로
    label = "accounts"                                  # ← label 짧게 (migration 등에 사용)
```

`apps/memos/apps.py`:
```python
class MemosConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "apps.memos"
    label = "memos"
```

> **`name`** = Python import path. **`label`** = 짧은 ID (default 는 마지막 segment). app 을 `apps/X` 에 두면 `name="apps.X"` 명시 필수.

---

## 7. URL routing — 분리

`config/urls.py`:
```python
from django.contrib import admin
from django.urls import path, include


urlpatterns = [
    path("admin/", admin.site.urls),
    path("api/auth/", include("apps.accounts.urls")),
    path("api/memos/", include("apps.memos.urls")),
]
```

`apps/memos/urls.py`:
```python
from django.urls import path, include
from rest_framework.routers import DefaultRouter

from .views import MemoViewSet


router = DefaultRouter()
router.register("", MemoViewSet, basename="memo")

urlpatterns = [
    path("", include(router.urls)),
]
```

→ Router 가 `GET /` / `POST /` / `GET /{id}/` / `PATCH /{id}/` / `DELETE /{id}/` 자동.

---

## 8. logging 설정

```python
# config/settings/base.py
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "verbose": {
            "format": "{asctime} [{levelname}] {name}: {message}",
            "style": "{",
        },
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "verbose",
        },
    },
    "root": {"handlers": ["console"], "level": "INFO"},
    "loggers": {
        "django": {"handlers": ["console"], "level": "INFO", "propagate": False},
        "django.db.backends": {"level": "WARNING"},     # SQL log off
        "apps": {"level": "DEBUG" if DEBUG else "INFO"},
    },
}
```

자세한 JSON / request_id 는 [[09-logging-mdc]].

---

## 9. 환경별 settings 사용 — 결정 시점

| 환경 | DJANGO_SETTINGS_MODULE |
| --- | --- |
| local dev | `config.settings.dev` |
| pytest | `config.settings.test` |
| staging | `config.settings.prod` (다른 .env) |
| prod | `config.settings.prod` |

```bash
# pytest
DJANGO_SETTINGS_MODULE=config.settings.test pytest
# 또는 pyproject.toml 에
```

`pyproject.toml`:
```toml
[tool.pytest.ini_options]
DJANGO_SETTINGS_MODULE = "config.settings.test"
```

---

## 10. 자주 막히는 함정

### 함정 1 — settings 단일 파일 + `if DEBUG` 분기 폭발
실수로 dev key 가 prod 에 들어감. **settings 패키지 분리**.

### 함정 2 — `apps/X` 에 두고 `apps.py` 의 `name` 미수정
`AppRegistryError`. **`name="apps.X"`** 명시.

### 함정 3 — `INSTALLED_APPS` 의 순서
admin / auth / contenttypes / sessions / messages / staticfiles 가 먼저. 3rd party / local 은 그 뒤.

### 함정 4 — `.env` 가 git commit
secret 노출. **`.gitignore`** + `.env.example` 만 commit.

### 함정 5 — `django-environ` 없이 `os.environ.get`
default fallback 부주의로 prod 가 dev key 로. django-environ 권장.

### 함정 6 — `DATABASE_URL` 파싱 실패
`env.db("DATABASE_URL")` 는 `mysql://...` / `postgres://...` 같은 URL 만. raw dict 면 `env.dict` 또는 직접.

### 함정 7 — `AUTH_USER_MODEL` 늦게 추가
migrate 후 변경 = 매우 복잡. **첫 migrate 전에 Custom User** ([[03-models-and-orm#5 Custom User|03 §5]]).

### 함정 8 — `DJANGO_SETTINGS_MODULE` env 누락
`manage.py` 가 default 로 dev 인데 prod 에서 default 가 prod 이 아니면 dev settings 로 prod 실행 = 재앙. **운영 명시**.

### 함정 9 — settings 의 `from .base import *`
`# noqa` 안 적으면 linter 가 unused import. `# noqa` 또는 `__all__`.

### 함정 10 — `DEBUG=False` + static 안 모음
`runserver` 가 static serve 안 함. **`collectstatic`** + Nginx serve.

---

## 11. 다음

→ [[03-models-and-orm]] — Model / migration / Custom User / `select_related`
