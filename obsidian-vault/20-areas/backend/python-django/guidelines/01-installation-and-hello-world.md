---
title: "01. Django 설치 + startproject + runserver"
kind: guide
status: current
created_at: 2026-05-26T10:10:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-django, guidelines, installation, hello-world]
---

# 01. Django 설치 + startproject + runserver

**[[guidelines|↑ Hub]]** · → 다음 [[02-project-structure]]

---

## 1. Python 준비

```bash
python3 --version            # 3.10+ (3.12 권장)
```

없으면:
- macOS: `brew install python@3.12`
- Ubuntu: `sudo apt install python3.12 python3.12-venv`

---

## 2. 프로젝트 디렉토리 + venv

```bash
mkdir memo-django && cd memo-django
python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
python --version                   # venv 안의 python
```

> **venv 안 만들면** pip 설치가 시스템 Python 을 오염. 프로젝트마다 venv.

---

## 3. Django 설치

```bash
pip install --upgrade pip
pip install django djangorestframework
pip freeze > requirements.txt
```

`requirements.txt`:
```
django==5.0.6
djangorestframework==3.15.2
```

> 본 가이드: Django **5.0+**, DRF **3.15+**. 이전 버전은 일부 syntax 다름.

---

## 4. startproject — 프로젝트 골격 생성

```bash
django-admin startproject config .
```

> 마지막 `.` = **현재 디렉토리에 생성** (config 라는 wrapping 폴더 X).

생성 결과:
```
memo-django/
├── manage.py                       # CLI 진입점
├── requirements.txt
└── config/                          # 프로젝트 settings 패키지
    ├── __init__.py
    ├── asgi.py                      # ASGI 진입점 (async / Channels)
    ├── settings.py                  # 모든 설정
    ├── urls.py                      # 최상단 URL routing
    └── wsgi.py                      # WSGI 진입점 (gunicorn)
```

> 관행: 프로젝트 이름을 `config` 또는 `memo` 로. **`mysite`** 같은 default 는 의미 없음.

---

## 5. 첫 migration + 실행

```bash
python manage.py migrate         # 내장 app (auth, admin, sessions) migration
python manage.py runserver       # 0.0.0.0:8000
```

```
System check identified no issues (0 silenced).
May 26, 2026 - 10:00:00
Django version 5.0.6, using settings 'config.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

브라우저 <http://localhost:8000> → "The install worked successfully!" 로켓.

---

## 6. admin superuser 생성 — 강력한 강점

```bash
python manage.py createsuperuser
# Username: admin
# Email: admin@example.com
# Password: ********
```

<http://localhost:8000/admin/> → User / Group 관리 자동.

> Django 의 **batteries-included** 의 정수. Flask / FastAPI 는 admin 도 별도 라이브러리 / 수동.

---

## 7. 첫 app — `startapp`

```bash
python manage.py startapp memos
```

생성:
```
memos/
├── __init__.py
├── admin.py                # admin 등록
├── apps.py                 # app config
├── migrations/
│   └── __init__.py
├── models.py               # 모델 (ORM)
├── tests.py                # test
└── views.py                # view (controller)
```

`config/settings.py` 의 `INSTALLED_APPS` 에 추가:

```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "rest_framework",                # DRF
    "memos",                          # 추가
]
```

---

## 8. 첫 view — 평범 Django

```python
# memos/views.py
from django.http import JsonResponse


def ping(request):
    return JsonResponse({"status": "ok", "message": "hello django"})
```

```python
# config/urls.py
from django.contrib import admin
from django.urls import path

from memos.views import ping


urlpatterns = [
    path("admin/", admin.site.urls),
    path("api/ping/", ping),
]
```

```bash
curl http://localhost:8000/api/ping/
# {"status": "ok", "message": "hello django"}
```

---

## 9. DRF view — 본 가이드의 표준

```python
# memos/views.py
from rest_framework.decorators import api_view
from rest_framework.response import Response


@api_view(["GET"])
def ping(request):
    return Response({
        "ok": True,
        "data": {"message": "hello django + drf"},
        "error": None,
    })
```

<http://localhost:8000/api/ping/> → DRF 의 **Browsable API** (브라우저 UI) + JSON 모두.

> DRF 의 강점: Browsable API. 개발 중 endpoint 를 브라우저에서 즉시 테스트.

---

## 10. `DEBUG=True` 의 의미

```python
# config/settings.py
DEBUG = True
```

- 에러 시 stack trace + local var + SQL 까지 노출 (사람용)
- `runserver` 가 static / media 자동 serve
- ALLOWED_HOSTS 검증 완화

> **운영 절대 X.** [[02-project-structure#9 환경별 settings|02 §9]] 의 env 분리.

---

## 11. 자주 막히는 함정

### 함정 1 — venv 활성화 안 하고 `pip install`
시스템 Python 오염. **항상 `source .venv/bin/activate` 먼저**.

### 함정 2 — `startproject mysite` (마지막 `.` 누락)
디렉토리 안에 또 디렉토리 (`memo-django/mysite/mysite/`). 마지막 `.` 권장.

### 함정 3 — `INSTALLED_APPS` 에 추가 안 함
새 app 의 model / migration / admin 인식 X.

### 함정 4 — `migrate` 안 하고 admin 접속
auth_user 테이블 X → admin 로그인 fail. **`migrate` 먼저**.

### 함정 5 — `DEBUG=True` 운영
stack trace / SQL 노출 = 정보 유출. **prod 반드시 False**.

### 함정 6 — `ALLOWED_HOSTS=[]` + `DEBUG=False`
모든 요청 400 Bad Request. **`ALLOWED_HOSTS=["api.example.com"]`** 명시.

### 함정 7 — `python manage.py` 가 venv 의 python 이 아님
시스템 python 으로 실행 → import error. **venv 활성화 확인** (`which python`).

### 함정 8 — `requirements.txt` 의 정확한 버전 누락
`django` 만 적으면 다음 사람이 6.x 설치. **`django==5.0.6`** 명시.

### 함정 9 — `createsuperuser` 의 weak password
경고 후 그대로 진행. **dev OK**. prod 는 강력하게.

---

## 12. 다음

→ [[02-project-structure]] — settings 분리 / app 분리 / requirements / .env
