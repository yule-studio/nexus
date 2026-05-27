---
title: "05. simple-jwt 인증 + 권한 + gunicorn + Docker 배포"
kind: guide
status: current
created_at: 2026-05-26T11:25:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-django, guidelines, jwt, simplejwt, deployment, docker]
---

# 05. simple-jwt 인증 + 권한 + gunicorn + Docker 배포

**[[guidelines|↑ Hub]]** ← [[04-views-and-serializers]] · → 다음 [[06-swagger-drf-spectacular]]

> Django 의 인증은 default = session/cookie. JSON API 는 **JWT (Bearer token)** = `djangorestframework-simplejwt`. 운영 = gunicorn + Nginx + Docker.

---

## 1. simple-jwt 설치

```bash
pip install djangorestframework-simplejwt
```

`settings/base.py`:
```python
from datetime import timedelta

INSTALLED_APPS += ["rest_framework_simplejwt"]

REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": (
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ),
    "DEFAULT_PERMISSION_CLASSES": (
        "rest_framework.permissions.IsAuthenticated",
    ),
    ...
}

SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=30),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=7),
    "ALGORITHM": "HS256",
    "SIGNING_KEY": env("JWT_SIGNING_KEY"),
    "AUTH_HEADER_TYPES": ("Bearer",),
    "USER_ID_FIELD": "id",
    "USER_ID_CLAIM": "user_id",
    "ROTATE_REFRESH_TOKENS": False,                       # True 면 refresh 마다 새 refresh 발급
    "BLACKLIST_AFTER_ROTATION": True,
    "LEEWAY": 30,                                          # 시각 오차 30초 허용
}
```

---

## 2. login / refresh — simple-jwt 의 built-in view

```python
# apps/accounts/urls.py
from django.urls import path
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView

from .views import SignupView


urlpatterns = [
    path("signup/", SignupView.as_view()),
    path("login/", TokenObtainPairView.as_view()),
    path("refresh/", TokenRefreshView.as_view()),
]
```

```bash
# login
curl -X POST http://localhost:8000/api/auth/login/ \
    -H 'Content-Type: application/json' \
    -d '{"email":"yuchan@example.com","password":"P@ssw0rd1!"}'
```

응답:
```json
{
    "access": "eyJhbGciOi...",
    "refresh": "eyJhbGciOi..."
}
```

> **`USERNAME_FIELD = "email"`** ([[03-models-and-orm#5 Custom User|03 §5]]) 이므로 login 의 입력 key 가 `email` 이 됨.

---

## 3. login 응답을 envelope 로 wrap

```python
# apps/accounts/views.py
from rest_framework_simplejwt.views import TokenObtainPairView
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer
from rest_framework.response import Response


class EmailTokenSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        token["email"] = user.email
        token["name"] = user.name
        return token


class LoginView(TokenObtainPairView):
    serializer_class = EmailTokenSerializer

    def post(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        try:
            serializer.is_valid(raise_exception=True)
        except Exception as e:
            return Response(
                {"ok": False, "data": None,
                 "error": {"code": "UNAUTHORIZED", "message": str(e)}},
                status=401,
            )
        data = serializer.validated_data
        return Response({
            "ok": True,
            "data": {
                "accessToken": data["access"],
                "refreshToken": data["refresh"],
                "tokenType": "Bearer",
            },
            "error": None,
        })
```

---

## 4. signup view

```python
# apps/accounts/serializers.py
from rest_framework import serializers
from .models import User


class SignupSerializer(serializers.Serializer):
    email = serializers.EmailField()
    password = serializers.CharField(min_length=8, write_only=True)
    name = serializers.CharField(max_length=50)

    def validate_email(self, value: str) -> str:
        if User.objects.filter(email=value).exists():
            raise serializers.ValidationError("이미 가입된 이메일")
        return value
```

```python
# apps/accounts/views.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from rest_framework.permissions import AllowAny

from .models import User
from .serializers import SignupSerializer


class SignupView(APIView):
    permission_classes = [AllowAny]                       # 공개

    def post(self, request):
        serializer = SignupSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        user = User.objects.create_user(**serializer.validated_data)
        return Response(
            {"ok": True,
             "data": {"id": user.id, "email": user.email, "name": user.name},
             "error": None},
            status=status.HTTP_201_CREATED,
        )
```

---

## 5. 요청 시 Bearer 토큰

```bash
curl http://localhost:8000/api/memos/ \
    -H 'Authorization: Bearer eyJhbGciOi...'
```

→ DRF 가 `JWTAuthentication` 으로 자동 검증 → `request.user` 에 User 인스턴스.

---

## 6. 권한 — Permission class

### 6.1 built-in

```python
from rest_framework.permissions import (
    AllowAny, IsAuthenticated, IsAuthenticatedOrReadOnly, IsAdminUser,
)


class PublicView(APIView):
    permission_classes = [AllowAny]


class MemoViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]
```

### 6.2 custom — IsOwner

```python
# apps/common/permissions.py
from rest_framework.permissions import BasePermission


class IsOwner(BasePermission):
    """`obj.user == request.user` 만 허용."""

    def has_object_permission(self, request, view, obj) -> bool:
        return obj.user == request.user
```

```python
class MemoViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated, IsOwner]
```

> `has_permission` = view-level / `has_object_permission` = object-level (DRF 가 `get_object` 후 호출).

### 6.3 role-based

```python
class Role:
    USER = "USER"
    EDITOR = "EDITOR"
    ADMIN = "ADMIN"


class HasRole(BasePermission):
    required_role: str = Role.USER

    def has_permission(self, request, view) -> bool:
        return request.user.is_authenticated and request.user.role == self.required_role


class IsAdmin(HasRole):
    required_role = Role.ADMIN
```

---

## 7. password hashing — Django default

Django default = PBKDF2 (4 ~ 5만 iteration). Argon2 가 더 강력:

```bash
pip install argon2-cffi
```

`base.py`:
```python
PASSWORD_HASHERS = [
    "django.contrib.auth.hashers.Argon2PasswordHasher",
    "django.contrib.auth.hashers.PBKDF2PasswordHasher",       # fallback
    "django.contrib.auth.hashers.BCryptSHA256PasswordHasher",
]
```

→ `User.set_password(...)` / `check_password(...)` 가 Argon2 사용 (첫 hasher).

---

## 8. CORS — django-cors-headers

```bash
pip install django-cors-headers
```

```python
INSTALLED_APPS += ["corsheaders"]
MIDDLEWARE = [
    "corsheaders.middleware.CorsMiddleware",              # security 다음
    "django.middleware.security.SecurityMiddleware",
    ...
]

CORS_ALLOWED_ORIGINS = env.list("CORS_ALLOWED_ORIGINS", default=[])
CORS_ALLOW_CREDENTIALS = True
```

> **`CORS_ALLOW_ALL_ORIGINS=True` 운영 X.** 정확한 origin 화이트리스트.

---

## 9. gunicorn — WSGI 서버

```bash
pip install gunicorn
```

```bash
DJANGO_SETTINGS_MODULE=config.settings.prod \
    gunicorn config.wsgi:application \
    --bind 0.0.0.0:8000 \
    --workers 4 \
    --threads 2 \
    --timeout 30 \
    --access-logfile - \
    --error-logfile -
```

| 옵션 | 의미 |
| --- | --- |
| `--workers N` | process 수 (CPU 코어 × 2 + 1 권장) |
| `--threads M` | worker 당 thread (mixed I/O) |
| `--timeout T` | request timeout (초) |
| `--bind` | host:port |
| `--access-logfile -` | stdout (k8s/docker 친화) |

→ async view (Django 4.0+) = `--worker-class uvicorn.workers.UvicornWorker`.

---

## 10. Nginx 앞단

```nginx
upstream django {
    server memo-api:8000;
}

server {
    listen 80;
    server_name api.example.com;

    client_max_body_size 10M;

    location /static/ {
        alias /app/staticfiles/;
    }

    location /media/ {
        alias /app/media/;
    }

    location / {
        proxy_pass http://django;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

→ static / media 는 Nginx 가 직접 serve. Django 는 동적 요청만.

---

## 11. Docker

`Dockerfile`:
```dockerfile
FROM python:3.12-slim AS base

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    DJANGO_SETTINGS_MODULE=config.settings.prod

WORKDIR /app

# system deps
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential default-libmysqlclient-dev pkg-config \
    && rm -rf /var/lib/apt/lists/*

# python deps
COPY requirements/ ./requirements/
RUN pip install --no-cache-dir -r requirements/prod.txt

# code
COPY . .

# static
RUN python manage.py collectstatic --noinput

EXPOSE 8000
CMD ["gunicorn", "config.wsgi:application", \
     "--bind", "0.0.0.0:8000", \
     "--workers", "4", "--threads", "2", \
     "--access-logfile", "-", "--error-logfile", "-"]
```

`docker-compose.yml`:
```yaml
services:
  api:
    build: .
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
    ports:
      - "8000:8000"
    command: >
      sh -c "python manage.py migrate &&
             gunicorn config.wsgi:application --bind 0.0.0.0:8000 --workers 4"

  db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: memo
      MYSQL_USER: memo
      MYSQL_PASSWORD: memo
      MYSQL_ROOT_PASSWORD: root
    healthcheck:
      test: ["CMD", "mysqladmin", "ping"]
      interval: 5s
      retries: 10
    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  mysql_data:
```

```bash
docker compose up -d --build
```

---

## 12. 운영 체크리스트

- [ ] `DEBUG=False`
- [ ] `ALLOWED_HOSTS` 명시
- [ ] `SECRET_KEY` env 에 (commit 금지)
- [ ] `DATABASE_URL` env
- [ ] `SECURE_SSL_REDIRECT=True` (HTTPS)
- [ ] `SESSION_COOKIE_SECURE=True`
- [ ] `CSRF_COOKIE_SECURE=True`
- [ ] `SECURE_HSTS_SECONDS=31536000`
- [ ] `python manage.py check --deploy` 통과
- [ ] `collectstatic` 실행
- [ ] migration 실행 (CD)
- [ ] healthz endpoint 노출 ([[08-monitoring]])
- [ ] Sentry / log 수집

---

## 13. 자주 막히는 함정

### 함정 1 — Custom User 의 `USERNAME_FIELD = "email"` → login 의 입력
`username` 이 아니라 `email` key. Postman 등 확인.

### 함정 2 — `IsAuthenticated` 가 default 인데 `permission_classes = []` 빼먹음
공개 endpoint 가 401. **`[AllowAny]`** 명시.

### 함정 3 — `IsOwner` 의 `has_object_permission` 만 → list 에 미적용
list endpoint 는 `get_queryset` 에서 user filtering. object permission 은 retrieve / update / delete 만.

### 함정 4 — JWT `SIGNING_KEY` 가 git
secret 노출. env.

### 함정 5 — refresh token 무한 사용
`ROTATE_REFRESH_TOKENS=True` + `BLACKLIST_AFTER_ROTATION=True` + `token_blacklist` app 으로 회전.

### 함정 6 — `argon2-cffi` 없는데 Argon2 hasher 명시
ImportError. **`pip install argon2-cffi`**.

### 함정 7 — gunicorn `--workers 1`
운영 = 처리량 부족. CPU 코어 × 2 + 1 권장.

### 함정 8 — Docker 의 `runserver` (개발 서버)
`runserver` 는 dev only. **gunicorn** 사용.

### 함정 9 — `collectstatic` 안 실행 + `DEBUG=False`
admin / DRF browsable 의 CSS 깨짐. **build 시 `collectstatic`**.

### 함정 10 — migration CD 누락
새 schema 가 DB 에 반영 X → 런타임 에러. **deploy 마다 `migrate`**.

### 함정 11 — `mysqlclient` 빌드 fail
system 의 `default-libmysqlclient-dev` 필요. Dockerfile 의 apt 단계.

### 함정 12 — `ALLOWED_HOSTS=["*"]` 운영
Host header 기반 공격 면역 없음. 정확한 도메인.

---

## 14. 다음

✅ 입문 (01-05) 끝. 운영 가능한 메모 API 완성.

기본 세팅 (06-10):
→ [[06-swagger-drf-spectacular]] — OpenAPI / Swagger UI / Authorize
→ [[07-viewset-patterns]] — ModelViewSet / mixin / filter / pagination 패턴
→ [[08-monitoring]] — django-prometheus + healthz
→ [[09-logging-mdc]] — django-log-request-id + JSON log
→ [[10-devtools-hotreload]] — runserver / pytest-django / debug-toolbar
