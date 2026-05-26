---
title: "Django + DRF Getting Started — 0 부터 첫 프로젝트 배포까지"
kind: guide
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T21:00:00+09:00
tags:
  - backend
  - python-django
  - drf
  - getting-started
  - guide
  - tutorial
---

# Django + DRF Getting Started — 0 부터 첫 프로젝트 배포까지

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | Django 5 + DRF 3.15 + 메모 API + 배포 단일 가이드 |

**[[python-django|↑ Django]]** · **[[../python-flask/getting-started|↗ Flask]]** · **[[../python-fastapi/getting-started|↗ FastAPI]]**

> **이 가이드를 보고 처음부터 Django + DRF 프로젝트를 만들고 배포까지 할 수 있도록 설계.**
> [[../python-flask/getting-started|Flask]] / [[../python-fastapi/getting-started|FastAPI]] 와 **같은 메모 API** 를 Django + Django REST Framework 로 — 세 가이드 비교하면 framework 차이 명확.
> 사전 지식: Python 기본 + HTTP/JSON/DB 약간.

---

## 0. Django 가 무엇인가

**Django** = Python 의 **full-stack 웹 프레임워크** (2005+). "batteries included" — ORM / admin / auth / template / forms / migration 모두 내장.

```python
# settings.py 하나로 모든 설정
# urls.py 로 라우팅
# models.py 로 DB
# views.py 로 HTTP 처리
# 4 개 파일만 만지면 끝
```

Java/Spring 진영의 **Spring Boot starter** 와 가장 가까움 — auto-configuration + 관습 over 설정.

### Django 의 4가지 핵심 특징

1. **MTV (Model-Template-View)** — MVC 의 Django 식. Model (ORM) / Template (HTML) / View (request handler).
2. **Django ORM** — SQLAlchemy 보다 더 단순, 더 마법. `User.objects.filter(email=...)` 직관적.
3. **Django Admin** — 모델 정의만 하면 `/admin/` 에 자동 CRUD UI. SaaS 운영자 화면 0줄 코드.
4. **Migration 내장** — `makemigrations` + `migrate`. Alembic / Flyway 별도 필요 X.

### Flask / FastAPI / Django — 한 줄

| | Flask | FastAPI | Django |
| --- | --- | --- | --- |
| 컨셉 | sync micro | async modern | **sync full-stack** |
| 기본 포함 | route only | route + pydantic + Swagger | route + ORM + admin + auth + template + migration |
| ORM | 별도 (Flask-SQLAlchemy) | 별도 (SQLAlchemy async) | **내장 (Django ORM)** |
| Migration | 별도 (Flask-Migrate / Alembic) | 별도 (Alembic) | **내장 (`makemigrations` / `migrate`)** |
| Admin UI | 없음 | 없음 | **내장** (`/admin/`) |
| API 만들기 | 직접 | 기본 | **Django REST Framework (DRF) 추가** |
| 학습 곡선 | 가장 쉬움 | 중간 | 가장 가파름 (관습 많음) |
| 적합 | 작은 API / 학습 | 신규 마이크로서비스 | 큰 SaaS / admin 필요 / 한국 SI |
| Spring 비유 | Spring MVC raw | Spring WebFlux | **Spring Boot full-stack starter** |

### 0.1 무엇을 만들 건가

Flask / FastAPI 가이드와 같은 **메모 API** — Django + DRF 로:

```
GET    /api/v1/memos/               # 목록
POST   /api/v1/memos/               # 생성
GET    /api/v1/memos/{id}/          # 단건
PATCH  /api/v1/memos/{id}/          # 수정
DELETE /api/v1/memos/{id}/          # 삭제

POST   /api/v1/auth/signup/
POST   /api/v1/auth/login/          # JWT 발급

# 보너스: Django admin
/admin/                              # admin UI 자동
```

> **Django URL convention**: 항상 끝 `/`. 안 붙이면 redirect 또는 404.

---

## 1. 사전 준비

### 1.1 Python 3.12 + venv

```bash
mkdir django-memo-api && cd django-memo-api
python3.12 -m venv .venv
source .venv/bin/activate
```

(자세한 설치는 [[../python-flask/getting-started#1 사전 준비|Flask §1]] 참고)

### 1.2 PostgreSQL — Flask 와 동일

```bash
docker run -d --name memo-db \
  -p 5432:5432 \
  -e POSTGRES_USER=memo \
  -e POSTGRES_PASSWORD=memo \
  -e POSTGRES_DB=memo \
  postgres:16
```

---

## 2. Django 설치 + 첫 프로젝트

### 2.1 의존성

```bash
pip install --upgrade pip
pip install django djangorestframework djangorestframework-simplejwt psycopg[binary] python-dotenv argon2-cffi
pip freeze > requirements.txt
```

설치 패키지:
- **Django 5.x** — framework 자체
- **djangorestframework (DRF)** — REST API 위한 표준 라이브러리
- **djangorestframework-simplejwt** — JWT 인증
- **psycopg[binary]** — PostgreSQL 드라이버
- **argon2-cffi** — 비밀번호 해시
- **python-dotenv** — `.env` 로드

### 2.2 첫 프로젝트 생성

```bash
# 프로젝트 (config) 생성 — 현재 디렉터리에
django-admin startproject config .

# 첫 app 생성 (모듈 단위 — 모델 + view + url 묶음)
python manage.py startapp memos
python manage.py startapp accounts
```

```
django-memo-api/
├── manage.py                    # CLI 진입점
├── config/                      # project (settings 모음)
│   ├── __init__.py
│   ├── settings.py              # 모든 설정 (큰 파일)
│   ├── urls.py                  # root URL conf
│   ├── asgi.py
│   └── wsgi.py
├── memos/                       # app 1
│   ├── __init__.py
│   ├── admin.py
│   ├── apps.py
│   ├── models.py
│   ├── views.py
│   ├── tests.py
│   └── migrations/
└── accounts/                    # app 2
    ├── (동일 구조)
```

> **Project vs App**: Django 는 **하나의 project** (config) + **여러 app** (도메인 단위). Spring 에서 `module` 분리와 비슷하지만 더 강제적.

### 2.3 첫 실행

```bash
python manage.py runserver
# Starting development server at http://127.0.0.1:8000/
```

브라우저 → <http://127.0.0.1:8000> → "The install worked successfully!" 페이지.

### 2.4 핵심 명령어 (manage.py)

| 명령 | 설명 |
| --- | --- |
| `runserver` | dev server (auto-reload) |
| `makemigrations` | 모델 변경 → migration 파일 생성 |
| `migrate` | DB 에 migration 적용 |
| `createsuperuser` | admin 계정 생성 |
| `startapp NAME` | 새 app 생성 |
| `shell` | Django context Python REPL |
| `test` | unit test 실행 |
| `collectstatic` | static 파일 한 곳에 모음 (배포) |

---

## 3. Settings — `config/settings.py`

Django 의 **모든 설정** 이 한 파일. 큰 편이지만 명확.

### 3.1 `.env` + dotenv

```bash
# .env
SECRET_KEY=django-dev-secret-change-in-prod
JWT_SIGNING_KEY=django-jwt-secret
DEBUG=True
DATABASE_URL=postgres://memo:memo@localhost:5432/memo
ALLOWED_HOSTS=localhost,127.0.0.1
CORS_ORIGINS=http://localhost:3000
```

### 3.2 settings.py — 주요 부분

```python
# config/settings.py
import os
from datetime import timedelta
from pathlib import Path
from dotenv import load_dotenv

load_dotenv()

BASE_DIR = Path(__file__).resolve().parent.parent

# Security
SECRET_KEY = os.environ["SECRET_KEY"]
DEBUG = os.environ.get("DEBUG", "False") == "True"
ALLOWED_HOSTS = os.environ.get("ALLOWED_HOSTS", "").split(",")

# Application
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
    "corsheaders",
    # 우리 app
    "accounts",
    "memos",
]

MIDDLEWARE = [
    "corsheaders.middleware.CorsMiddleware",            # CORS 가 제일 위
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]

ROOT_URLCONF = "config.urls"

# Database
import dj_database_url      # pip install dj-database-url 권장

DATABASES = {
    "default": dj_database_url.parse(
        os.environ["DATABASE_URL"],
        conn_max_age=600,                              # connection pool
    ),
}

# 또는 직접 작성:
# DATABASES = {
#     "default": {
#         "ENGINE": "django.db.backends.postgresql",
#         "NAME": "memo",
#         "USER": "memo",
#         "PASSWORD": "memo",
#         "HOST": "localhost",
#         "PORT": "5432",
#     }
# }

# Custom user model (필수 — 처음부터)
AUTH_USER_MODEL = "accounts.User"

# Password hashing — Argon2 우선
PASSWORD_HASHERS = [
    "django.contrib.auth.hashers.Argon2PasswordHasher",
    "django.contrib.auth.hashers.PBKDF2PasswordHasher",
    "django.contrib.auth.hashers.BCryptSHA256PasswordHasher",
]

# DRF
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": (
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ),
    "DEFAULT_PERMISSION_CLASSES": (
        "rest_framework.permissions.IsAuthenticated",   # 기본 — 인증 필수
    ),
    "DEFAULT_RENDERER_CLASSES": (
        "rest_framework.renderers.JSONRenderer",
    ),
    "DEFAULT_PARSER_CLASSES": (
        "rest_framework.parsers.JSONParser",
    ),
    "EXCEPTION_HANDLER": "config.exceptions.custom_exception_handler",
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 20,
}

# Simple JWT
SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=15),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=14),
    "ROTATE_REFRESH_TOKENS": True,
    "BLACKLIST_AFTER_ROTATION": True,
    "ALGORITHM": "HS256",
    "SIGNING_KEY": os.environ.get("JWT_SIGNING_KEY", SECRET_KEY),
    "VERIFYING_KEY": None,
    "AUDIENCE": None,
    "ISSUER": "memo-api",
    "AUTH_HEADER_TYPES": ("Bearer",),
}

# CORS
CORS_ALLOWED_ORIGINS = os.environ.get("CORS_ORIGINS", "http://localhost:3000").split(",")
CORS_ALLOW_CREDENTIALS = False

# i18n / tz
LANGUAGE_CODE = "ko-kr"
TIME_ZONE = "UTC"
USE_I18N = True
USE_TZ = True

STATIC_URL = "static/"
DEFAULT_AUTO_FIELD = "django.db.models.BigAutoField"
```

### 3.3 root URL — `config/urls.py`

```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path("admin/", admin.site.urls),
    path("api/v1/auth/", include("accounts.urls")),
    path("api/v1/memos/", include("memos.urls")),
]
```

> **`include("app.urls")`** — app 의 URL 을 root 에 mount. Spring 의 `@RequestMapping("/api/v1")` 동등.

---

## 4. Custom User model — `accounts`

Django 의 기본 `User` 는 `username` + `password`. 우리는 **email** + password 가 필요 → **반드시 처음부터 custom User**.

```python
# accounts/models.py
from django.contrib.auth.models import AbstractBaseUser, BaseUserManager, PermissionsMixin
from django.db import models
from django.utils import timezone


class UserManager(BaseUserManager):
    def create_user(self, email: str, password: str, **extra) -> "User":
        if not email:
            raise ValueError("Email 필수")
        email = self.normalize_email(email)
        user = self.model(email=email, **extra)
        user.set_password(password)                    # Argon2 hash 자동
        user.save(using=self._db)
        return user

    def create_superuser(self, email: str, password: str, **extra) -> "User":
        extra.setdefault("is_staff", True)
        extra.setdefault("is_superuser", True)
        extra.setdefault("is_active", True)
        return self.create_user(email, password, **extra)


class User(AbstractBaseUser, PermissionsMixin):
    email = models.EmailField(unique=True, db_index=True)
    name = models.CharField(max_length=100)
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)
    created_at = models.DateTimeField(default=timezone.now)

    objects = UserManager()

    USERNAME_FIELD = "email"
    REQUIRED_FIELDS = ["name"]                          # createsuperuser 시 묻는 추가 필드

    class Meta:
        db_table = "users"

    def __str__(self) -> str:
        return self.email
```

> **`AbstractBaseUser` + `PermissionsMixin`** — Django 의 모든 인증 기능 (password / permission group / superuser) 활용하면서 필드만 커스텀.
> **`USERNAME_FIELD = "email"`** — login identifier.
> **`AUTH_USER_MODEL = "accounts.User"`** (settings.py) — Django 전체가 이 모델 사용.

---

## 5. Memo model — `memos`

```python
# memos/models.py
from django.db import models
from django.conf import settings


class Memo(models.Model):
    owner = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name="memos",
    )
    title = models.CharField(max_length=200)
    content = models.TextField(default="")
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        db_table = "memos"
        ordering = ["-created_at"]                      # default order
        indexes = [
            models.Index(fields=["owner", "-created_at"]),
        ]

    def __str__(self) -> str:
        return f"{self.title} by {self.owner.email}"
```

> **`auto_now_add` / `auto_now`** — INSERT 시점 / 매 UPDATE 시점 자동 설정. SQLAlchemy 의 `default=lambda: ...` 와 동등.

---

## 6. 첫 마이그레이션

```bash
python manage.py makemigrations
# Migrations for 'accounts':
#   accounts/migrations/0001_initial.py
#     - Create model User
# Migrations for 'memos':
#   memos/migrations/0001_initial.py
#     - Create model Memo

python manage.py migrate
# Operations to perform:
#   Apply all migrations: ...
# Running migrations:
#   Applying contenttypes.0001_initial... OK
#   ... (Django 내장 admin / auth 마이그레이션도 자동)
```

> **Migration 자동 생성** — 모델 diff 를 보고 Django 가 SQL 생성. 운영 적용 전 review 필수.

확인:
```bash
psql postgresql://memo:memo@localhost:5432/memo -c "\dt"
# auth_*, django_*, accounts_user, memos_memo, ...
```

### 6.1 superuser 생성

```bash
python manage.py createsuperuser
# Email: admin@memo.local
# Name: admin
# Password: ********
# Superuser created successfully.
```

### 6.2 admin 등록

```python
# accounts/admin.py
from django.contrib import admin
from .models import User

admin.site.register(User)


# memos/admin.py
from django.contrib import admin
from .models import Memo

@admin.register(Memo)
class MemoAdmin(admin.ModelAdmin):
    list_display = ("id", "title", "owner", "created_at")
    list_filter = ("created_at",)
    search_fields = ("title", "content")
```

→ `python manage.py runserver` → <http://localhost:8000/admin/> → email/password 로 login → User / Memo 자동 CRUD UI.

> **이게 Django 의 킬러 기능** — 한 줄도 코드 안 쓰고 admin 화면. SaaS 운영자 대시보드 즉시.

---

## 7. DRF Serializer — Request / Response

```python
# accounts/serializers.py
from rest_framework import serializers
from django.contrib.auth import authenticate
from rest_framework_simplejwt.tokens import RefreshToken
from .models import User


class SignupSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True, min_length=8, max_length=64)

    class Meta:
        model = User
        fields = ["id", "email", "name", "password"]
        read_only_fields = ["id"]

    def create(self, validated_data: dict) -> User:
        password = validated_data.pop("password")
        return User.objects.create_user(password=password, **validated_data)


class LoginSerializer(serializers.Serializer):
    email = serializers.EmailField()
    password = serializers.CharField(write_only=True)

    def validate(self, attrs: dict) -> dict:
        user = authenticate(
            request=self.context.get("request"),
            username=attrs["email"],
            password=attrs["password"],
        )
        if user is None:
            raise serializers.ValidationError({"detail": "이메일 또는 비밀번호가 올바르지 않습니다."})
        attrs["user"] = user
        return attrs


class TokenSerializer(serializers.Serializer):
    accessToken = serializers.CharField()
    refreshToken = serializers.CharField()
    tokenType = serializers.CharField(default="Bearer")
    expiresIn = serializers.IntegerField(default=900)


class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ["id", "email", "name", "created_at"]
        read_only_fields = ["id", "created_at"]
```

```python
# memos/serializers.py
from rest_framework import serializers
from .models import Memo


class MemoSerializer(serializers.ModelSerializer):
    createdAt = serializers.DateTimeField(source="created_at", read_only=True)
    updatedAt = serializers.DateTimeField(source="updated_at", read_only=True)

    class Meta:
        model = Memo
        fields = ["id", "title", "content", "createdAt", "updatedAt"]
        read_only_fields = ["id", "createdAt", "updatedAt"]


class MemoCreateSerializer(serializers.ModelSerializer):
    class Meta:
        model = Memo
        fields = ["title", "content"]


class MemoUpdateSerializer(serializers.ModelSerializer):
    class Meta:
        model = Memo
        fields = ["title", "content"]
        extra_kwargs = {
            "title": {"required": False},
            "content": {"required": False},
        }
```

> **`ModelSerializer`** — Model 의 field 를 자동으로 serializer field 로. Spring 의 `MapStruct` + Lombok 합친 느낌.
> **`write_only=True`** — request 만 받고 response 에 안 노출 (password).
> **`source="created_at"`** — DB column ↔ JSON field 이름 다를 때.

---

## 8. View / URL — accounts

```python
# accounts/views.py
from rest_framework import status
from rest_framework.generics import CreateAPIView, GenericAPIView
from rest_framework.permissions import AllowAny, IsAuthenticated
from rest_framework.response import Response
from rest_framework_simplejwt.tokens import RefreshToken

from .models import User
from .serializers import SignupSerializer, LoginSerializer, UserSerializer


class SignupView(CreateAPIView):
    queryset = User.objects.all()
    serializer_class = SignupSerializer
    permission_classes = [AllowAny]                     # 공개


class LoginView(GenericAPIView):
    serializer_class = LoginSerializer
    permission_classes = [AllowAny]

    def post(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data, context={"request": request})
        serializer.is_valid(raise_exception=True)
        user: User = serializer.validated_data["user"]

        refresh = RefreshToken.for_user(user)
        refresh["email"] = user.email
        refresh["name"] = user.name

        return Response(
            {
                "accessToken": str(refresh.access_token),
                "refreshToken": str(refresh),
                "tokenType": "Bearer",
                "expiresIn": 900,
            },
            status=status.HTTP_200_OK,
        )


class MeView(GenericAPIView):
    serializer_class = UserSerializer
    permission_classes = [IsAuthenticated]              # 인증 필수

    def get(self, request, *args, **kwargs):
        serializer = self.get_serializer(request.user)
        return Response(serializer.data)
```

```python
# accounts/urls.py
from django.urls import path
from .views import SignupView, LoginView, MeView

urlpatterns = [
    path("signup/", SignupView.as_view(), name="signup"),
    path("login/", LoginView.as_view(), name="login"),
    path("me/", MeView.as_view(), name="me"),
]
```

> **`CreateAPIView` / `GenericAPIView`** — DRF 의 class-based view. 표준 동작 (POST → create / serializer 검증 / 응답) 자동.
> **`permission_classes`** — endpoint 단 인증 강제. `IsAuthenticated` 가 default, 공개는 `AllowAny`.

---

## 9. ViewSet + Router — memos (가장 Django/DRF 스러운 방식)

```python
# memos/views.py
from rest_framework import status, viewsets, mixins
from rest_framework.exceptions import PermissionDenied
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response

from .models import Memo
from .serializers import MemoSerializer, MemoCreateSerializer, MemoUpdateSerializer


class MemoViewSet(viewsets.ModelViewSet):
    """
    list / create / retrieve / update / partial_update / destroy 5 action 자동 생성.
    Spring 의 @RestController + @RequestMapping + 5 action 한 클래스에.
    """

    queryset = Memo.objects.all()
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        # 본인 메모만
        return Memo.objects.filter(owner=self.request.user)

    def get_serializer_class(self):
        if self.action == "create":
            return MemoCreateSerializer
        if self.action in ("update", "partial_update"):
            return MemoUpdateSerializer
        return MemoSerializer

    def perform_create(self, serializer):
        # 작성자는 자동으로 현재 user
        serializer.save(owner=self.request.user)

    def create(self, request, *args, **kwargs):
        # response 는 MemoSerializer 로 직렬화
        create_ser = self.get_serializer(data=request.data)
        create_ser.is_valid(raise_exception=True)
        memo = Memo.objects.create(owner=request.user, **create_ser.validated_data)
        out = MemoSerializer(memo)
        return Response(out.data, status=status.HTTP_201_CREATED)
```

```python
# memos/urls.py
from rest_framework.routers import DefaultRouter
from .views import MemoViewSet

router = DefaultRouter()
router.register(r"", MemoViewSet, basename="memo")

urlpatterns = router.urls
```

이게 자동 생성하는 URL:
```
GET    /api/v1/memos/               → list
POST   /api/v1/memos/               → create
GET    /api/v1/memos/{pk}/          → retrieve
PUT    /api/v1/memos/{pk}/          → update
PATCH  /api/v1/memos/{pk}/          → partial_update
DELETE /api/v1/memos/{pk}/          → destroy
```

> **ViewSet + Router** = Spring 의 `@RestController` + `@RequestMapping` 풀세트. 한 클래스로 CRUD 5개 API 완성.

### 9.1 함수형 view (대안 — 더 단순)

ViewSet 이 어려우면 함수형도 OK:

```python
from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticated


@api_view(["GET", "POST"])
@permission_classes([IsAuthenticated])
def memo_list_or_create(request):
    if request.method == "GET":
        memos = Memo.objects.filter(owner=request.user)
        return Response(MemoSerializer(memos, many=True).data)
    if request.method == "POST":
        serializer = MemoCreateSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        memo = Memo.objects.create(owner=request.user, **serializer.validated_data)
        return Response(MemoSerializer(memo).data, status=201)
```

→ Flask blueprint 스타일. 함수 = endpoint. 본 가이드는 **ViewSet** 가 표준 (Django 진영 표준).

---

## 10. 글로벌 에러 핸들러

```python
# config/exceptions.py
import logging
from rest_framework.views import exception_handler
from rest_framework.response import Response
from rest_framework import status as http_status
from rest_framework.exceptions import (
    AuthenticationFailed, NotAuthenticated, PermissionDenied,
    NotFound, ValidationError, APIException,
)

log = logging.getLogger(__name__)


def _envelope(status: int, code: str, message: str, details=None) -> dict:
    body = {"success": False, "error": {"code": code, "message": message}}
    if details is not None:
        body["error"]["details"] = details
    return body


def custom_exception_handler(exc, context):
    """DRF default + 우리 envelope."""

    # DRF default 가 일단 response 만들어줌
    response = exception_handler(exc, context)

    if response is None:
        # 캐치되지 않은 예외 → 500
        log.exception("UNCAUGHT")
        return Response(
            _envelope(500, "INTERNAL_SERVER_ERROR", "서버 내부 오류가 발생했습니다."),
            status=500,
        )

    # 매핑
    if isinstance(exc, (NotAuthenticated, AuthenticationFailed)):
        data = _envelope(401, "UNAUTHORIZED", str(exc.detail) if hasattr(exc, "detail") else "인증이 필요합니다.")
    elif isinstance(exc, PermissionDenied):
        data = _envelope(403, "FORBIDDEN", "권한이 없습니다.")
    elif isinstance(exc, NotFound):
        data = _envelope(404, "NOT_FOUND", "리소스를 찾을 수 없습니다.")
    elif isinstance(exc, ValidationError):
        data = _envelope(422, "VALIDATION_ERROR", "입력 형식이 올바르지 않습니다.", exc.detail)
        response.status_code = 422
    elif isinstance(exc, APIException):
        data = _envelope(response.status_code, "API_ERROR", str(exc.detail) if hasattr(exc, "detail") else "API error")
    else:
        data = _envelope(response.status_code, "ERROR", str(exc))

    response.data = data
    return response
```

> Settings 의 `"EXCEPTION_HANDLER": "config.exceptions.custom_exception_handler"` 가 자동 연결.

---

## 11. 실습 — curl 시나리오

```bash
python manage.py runserver

# 1. signup
curl -X POST http://localhost:8000/api/v1/auth/signup/ \
  -H "Content-Type: application/json" \
  -d '{"email":"yuchan@example.com","password":"P@ssw0rd1!","name":"yuchan"}'
# 201 + user

# 2. login
TOKEN=$(curl -sX POST http://localhost:8000/api/v1/auth/login/ \
  -H "Content-Type: application/json" \
  -d '{"email":"yuchan@example.com","password":"P@ssw0rd1!"}' \
  | jq -r .accessToken)

# 3. me
curl http://localhost:8000/api/v1/auth/me/ -H "Authorization: Bearer $TOKEN"

# 4. 메모 CRUD
curl -X POST http://localhost:8000/api/v1/memos/ \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"첫 메모","content":"안녕하세요"}'

curl http://localhost:8000/api/v1/memos/ -H "Authorization: Bearer $TOKEN"

curl -X PATCH http://localhost:8000/api/v1/memos/1/ \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title":"수정됨"}'

curl -X DELETE http://localhost:8000/api/v1/memos/1/ -H "Authorization: Bearer $TOKEN"

# 5. admin 화면 — 브라우저에서
open http://localhost:8000/admin/
```

> Django URL convention — 항상 끝 `/`. `/memos` 가 아니라 `/memos/`.

---

## 12. 테스트 — Django test framework

Django 는 **자체 test framework** (pytest 도 OK).

```python
# memos/tests.py
from django.urls import reverse
from rest_framework import status
from rest_framework.test import APITestCase

from accounts.models import User
from memos.models import Memo


class MemoApiTests(APITestCase):

    @classmethod
    def setUpTestData(cls):
        cls.user = User.objects.create_user(
            email="t@example.com", password="P@ssw0rd1!", name="t",
        )
        cls.other = User.objects.create_user(
            email="o@example.com", password="P@ssw0rd1!", name="o",
        )

    def setUp(self):
        # JWT 발급 후 client 에 attach
        from rest_framework_simplejwt.tokens import RefreshToken
        token = RefreshToken.for_user(self.user).access_token
        self.client.credentials(HTTP_AUTHORIZATION=f"Bearer {token}")

    def test_create_and_list(self):
        # create
        res = self.client.post("/api/v1/memos/", {"title": "t1", "content": "c1"}, format="json")
        self.assertEqual(res.status_code, status.HTTP_201_CREATED)
        memo_id = res.data["id"]

        # list
        res = self.client.get("/api/v1/memos/")
        self.assertEqual(res.status_code, 200)
        self.assertEqual(len(res.data["results"]), 1)
        self.assertEqual(res.data["results"][0]["id"], memo_id)

    def test_cannot_access_others_memo(self):
        # 다른 user 의 메모 생성
        other_memo = Memo.objects.create(owner=self.other, title="secret", content="...")

        # 접근 — get_queryset 가 본인만 필터 → 404
        res = self.client.get(f"/api/v1/memos/{other_memo.id}/")
        self.assertEqual(res.status_code, 404)

    def test_validation_error(self):
        res = self.client.post("/api/v1/memos/", {"content": "title 누락"}, format="json")
        self.assertEqual(res.status_code, 422)
        self.assertIn("title", str(res.data))
```

```bash
python manage.py test
# Creating test database for alias 'default'...
# .....
# Ran 3 tests in 0.5s
# OK
```

> **`APITestCase`** — DRF 의 self.client 자동 + DB transaction 마다 rollback.
> **`setUpTestData`** — 클래스 단위 1회만 (모든 test 공유). `setUp` 은 각 test 마다.

---

## 13. 배포 — gunicorn + Docker + Nginx

### 13.1 gunicorn

```bash
pip install gunicorn

gunicorn config.wsgi:application \
  --workers 4 \
  --bind 0.0.0.0:8000 \
  --access-logfile -
```

### 13.2 static 파일 (배포 시 필수)

```bash
python manage.py collectstatic --noinput
# STATIC_ROOT 에 모든 static (admin CSS 등) 모임
```

```python
# config/settings.py
STATIC_URL = "static/"
STATIC_ROOT = BASE_DIR / "staticfiles"             # collectstatic 의 결과
```

→ Nginx 가 `/static/` 을 직접 서빙. Django 는 `/static/` 처리 X.

### 13.3 Dockerfile

```dockerfile
FROM python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
        libpq5 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# static 수집
RUN python manage.py collectstatic --noinput || true

RUN useradd -m -u 1000 django && chown -R django:django /app
USER django

ENV PORT=8000
EXPOSE 8000

CMD ["sh", "-c", "python manage.py migrate --noinput && gunicorn config.wsgi:application --workers 4 --bind 0.0.0.0:8000 --access-logfile -"]
```

### 13.4 docker-compose

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: memo
      POSTGRES_PASSWORD: memo
      POSTGRES_DB: memo
    ports: ["5432:5432"]
    volumes: ["dbdata:/var/lib/postgresql/data"]

  app:
    build: .
    environment:
      SECRET_KEY: dev-secret
      JWT_SIGNING_KEY: dev-jwt
      DEBUG: "False"
      DATABASE_URL: postgres://memo:memo@db:5432/memo
      ALLOWED_HOSTS: "*"
    ports: ["8000:8000"]
    depends_on: [db]

volumes:
  dbdata:
```

### 13.5 운영 체크리스트

- [ ] `DEBUG = False` 운영 — 절대 True 금지 (debug page = 코드 노출)
- [ ] `SECRET_KEY` 가 vault (env 직접 X — DJANGO_SECRET_KEY env)
- [ ] `ALLOWED_HOSTS` 가 명시적 (와일드카드 신중)
- [ ] HTTPS (Nginx / ALB TLS termination)
- [ ] `SECURE_SSL_REDIRECT = True`, `SECURE_HSTS_SECONDS = 31536000` 등 보안 헤더
- [ ] `python manage.py migrate --noinput` 가 컨테이너 startup
- [ ] `collectstatic` 가 build time
- [ ] Nginx 가 `/static/` 직접 서빙
- [ ] PostgreSQL `max_connections` ≥ workers × conn_max_age
- [ ] healthcheck — django-health-check 또는 직접 view
- [ ] Sentry / log → stdout

```python
# config/urls.py 에
from django.http import JsonResponse
def healthz(_request):
    return JsonResponse({"status": "ok"})

urlpatterns = [
    # ...
    path("healthz/", healthz),
]
```

---

## 14. Django ORM — 자주 쓰는 패턴

```python
# 단건
user = User.objects.get(email="a@b.com")          # 없으면 DoesNotExist
user = User.objects.filter(email="a@b.com").first()  # 없으면 None

# 목록
memos = Memo.objects.filter(owner=user, title__icontains="검색어").order_by("-created_at")

# 페이지네이션 (raw — DRF Pagination 권장)
memos = Memo.objects.all()[0:20]                  # LIMIT 20 OFFSET 0

# 집계
from django.db.models import Count, Avg
stats = Memo.objects.filter(owner=user).aggregate(total=Count("id"))
# {"total": 42}

# 일괄 update
Memo.objects.filter(owner=user).update(title="bulk")  # SQL 한 번

# JOIN — select_related (FK / OneToOne)
memos = Memo.objects.select_related("owner").all()
# JOIN users — N+1 회피

# JOIN — prefetch_related (Many)
users = User.objects.prefetch_related("memos").all()
# 별도 query 로 메모들 미리

# Q object — OR 조건
from django.db.models import Q
Memo.objects.filter(Q(title="x") | Q(content__icontains="y"))

# Transaction
from django.db import transaction

with transaction.atomic():
    user.save()
    memo.save()
    # 한쪽 실패 시 모두 rollback
```

> **함정 — N+1**: `Memo.objects.all()` 후 `for m in memos: print(m.owner.email)` = 각 m 마다 user query. **`select_related("owner")`** 로 JOIN.

---

## 15. 자주 막히는 함정

### 함정 1 — Custom User 안 만들고 시작
프로젝트 깊어진 뒤 `AUTH_USER_MODEL` 변경 = 사실상 불가. **첫 migration 전에 반드시 custom User**.

### 함정 2 — `DEBUG = True` 운영
debug page 가 코드 / 환경변수 노출. **운영 절대 False**.

### 함정 3 — `ALLOWED_HOSTS` 비어있음
`DEBUG=False` 면 모든 요청이 400 (DisallowedHost). **`ALLOWED_HOSTS=*` (개발) 또는 도메인 명시**.

### 함정 4 — Migration 충돌
여러 dev 가 동시에 `makemigrations` → 같은 number prefix (예: 0003) 충돌. **`python manage.py makemigrations --name X` + PR review** 또는 `migrate --merge`.

### 함정 5 — URL 끝 `/` 누락
`/memos` 호출 시 Django 가 `/memos/` 로 redirect (`APPEND_SLASH=True`). POST 의 경우 redirect = body 잃음. **클라가 `/` 포함 보내기**.

### 함정 6 — `select_related` vs `prefetch_related` 혼동
- `select_related("owner")` — FK / OneToOne → SQL JOIN
- `prefetch_related("memos")` — Many → 별도 query

ManyToMany 에 `select_related` = 동작 X.

### 함정 7 — `objects.all()` 에 `len()` 호출
`len(qs)` = 모든 row fetch. count 만 필요하면 **`qs.count()`** (SQL `SELECT COUNT`).

### 함정 8 — `QuerySet` 의 lazy evaluation
`qs = User.objects.filter(...)` 만으로는 SQL 안 됨. `list(qs)`, `for u in qs`, `len(qs)`, indexing 등 평가 시점에 SQL.

### 함정 9 — DRF `Response(data)` 안에 ORM 객체 직접
serializer 안 거치면 직렬화 안 됨. **`Response(MemoSerializer(memo).data)`**.

### 함정 10 — `ModelSerializer` 의 `read_only_fields` 누락
ID / created_at 같은 자동 필드를 user 가 send 가능 → 위변조. **명시적 `read_only_fields`**.

### 함정 11 — `default_auto_field` 변경
Django 3.2+ default 가 `BigAutoField`. 옛 프로젝트는 `AutoField`. settings.py `DEFAULT_AUTO_FIELD` 명시.

### 함정 12 — CORS 가 admin 막음
`CORS_ALLOWED_ORIGINS` 가 admin 도 막을 수 있음 (FE 가 admin 호출 시). `CORS_URLS_REGEX = r"^/api/.*$"` 로 admin 제외.

---

## 16. 다음 학습 로드맵

| 단계 | 학습 | 노트 / 외부 |
| --- | --- | --- |
| 1 | 본 가이드 직접 따라 만들기 | (이 노트) |
| 2 | Django admin 깊이 (list_display / inline / action) | <https://docs.djangoproject.com/en/5.0/ref/contrib/admin/> |
| 3 | DRF Permission / Throttling / Authentication 깊이 | <https://www.django-rest-framework.org/api-guide/permissions/> |
| 4 | Django ORM 깊이 (Manager / aggregation / window function) | <https://docs.djangoproject.com/en/5.0/topics/db/queries/> |
| 5 | Celery + Redis (Django 와 통합) | <https://docs.celeryproject.org/en/stable/django/> |
| 6 | Django Channels (WebSocket) | <https://channels.readthedocs.io/> |
| 7 | Signals (모델 변경 이벤트 hook) | <https://docs.djangoproject.com/en/5.0/topics/signals/> |
| 8 | Security 강화 (CSP / HSTS / CSRF / SecurityMiddleware) | django security checklist |
| 9 | 다국어 (i18n) | <https://docs.djangoproject.com/en/5.0/topics/i18n/> |

---

## 17. 참고 / cross-link

- [[python-django|↑ Django area]]
- [[../python-flask/getting-started|↗ Flask 가이드]] — 비교
- [[../python-fastapi/getting-started|↗ FastAPI 가이드]] — 비교
- Django 공식: <https://www.djangoproject.com/>
- Django 5 docs: <https://docs.djangoproject.com/en/5.0/>
- Django REST Framework: <https://www.django-rest-framework.org/>
- Simple JWT: <https://django-rest-framework-simplejwt.readthedocs.io/>
- "Django for Beginners" (William Vincent) 책 권장
