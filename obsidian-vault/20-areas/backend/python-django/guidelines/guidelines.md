---
title: "Django Guidelines — 입문 + 기본 세팅"
kind: index
status: current
created_at: 2026-05-26T10:00:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-django, guidelines, getting-started]
---

# Django Guidelines — 입문 + 기본 세팅

**[[../python-django|↑ python-django]]**

> Django 5 + DRF + simple-jwt 로 "메모 API" 를 끝까지 만들면서 각 단계마다 **왜** 그렇게 하는지 + **자주 막히는 함정**.

---

## 학습 흐름

10 단계. 입문 5 + 기본 세팅 5.

### 입문 (01-05) — 메모 API 한 바퀴

1. [[01-installation-and-hello-world]] — Python / venv / Django / startproject / runserver
2. [[02-project-structure]] — `manage.py` / settings 분리 / app 분리 / requirements
3. [[03-models-and-orm]] — 모델 / migration / `select_related` / Custom User
4. [[04-views-and-serializers]] — DRF ViewSet / Serializer / Router / validation
5. [[05-auth-and-deployment]] — simple-jwt / 권한 / gunicorn + Nginx + Docker

### 기본 세팅 (06-10) — 운영 가능 수준

6. [[06-swagger-drf-spectacular]] — drf-spectacular OpenAPI / Swagger UI / Authorize
7. [[07-viewset-patterns]] — ModelViewSet / mixin / 권한 / filter / pagination
8. [[08-monitoring]] — django-prometheus + healthz
9. [[09-logging-mdc]] — django-log-request-id + JSON log
10. [[10-devtools-hotreload]] — runserver / pytest-django / debug-toolbar / black / ruff

---

## 메모 API — 한 줄 요구사항

회원가입 / 로그인 (이메일 + 비밀번호) → JWT → 자기 메모 CRUD (목록 / 단건 / 생성 / 수정 / 삭제).

| Method | URL | 인증 | 설명 |
| --- | --- | --- | --- |
| POST | `/api/auth/signup/` | X | 회원가입 |
| POST | `/api/auth/login/` | X | JWT 발급 |
| POST | `/api/auth/refresh/` | X (refresh token) | access 재발급 |
| GET | `/api/memos/` | O | 내 메모 목록 (page/page_size) |
| POST | `/api/memos/` | O | 생성 |
| GET | `/api/memos/{id}/` | O | 단건 |
| PATCH | `/api/memos/{id}/` | O | 부분 수정 |
| DELETE | `/api/memos/{id}/` | O | 삭제 |

> Django/DRF 관행: URL 끝 trailing slash (`/`). 본 가이드도 동일.

응답 envelope (FastAPI / Flask 와 동일 — 비교 용이):

```json
{ "ok": true, "data": { ... }, "error": null, "request_id": "..." }
```

---

## 다른 framework 와 비교

| 영역 | Spring | Flask | FastAPI | Django (← 본) |
| --- | --- | --- | --- | --- |
| 설치 | `start.spring.io` | `pip install flask` | `pip install "fastapi[standard]"` | `pip install django djangorestframework` |
| 진입점 | `@SpringBootApplication` | `Flask(__name__)` | `FastAPI()` | `manage.py runserver` |
| 라우팅 | `@RestController` | `Blueprint` | `APIRouter` | `urls.py` + `ViewSet` |
| Validation | Bean Validation | marshmallow | pydantic | **DRF Serializer** |
| ORM | JPA | SQLAlchemy | SQLAlchemy 2.x async | **Django ORM** |
| Migration | Flyway | Flask-Migrate | Alembic | **migrate** (자동) |
| DI | Spring | Flask-Injector / 수동 | `Depends()` | `apps`/`settings` (대부분 수동) |
| Auth | Spring Security | Flask-JWT-Extended | python-jose + OAuth2 | **simple-jwt** |
| API doc | springdoc-openapi | flask-smorest | 자동 | **drf-spectacular** |
| Admin | 별도 | 별도 | 별도 | **자동 (manage.py)** ← 강점 |
| Test | JUnit | pytest | pytest-asyncio | **pytest-django** |

---

## 학습 환경 권장

- Python 3.12 (3.10+ 권장)
- pip + venv (또는 uv)
- VS Code + Python extension + Pylance
- MySQL 8 또는 PostgreSQL 15 (학습은 sqlite 도 OK — Django 기본)
- HTTPie / Bruno / Postman / VS Code REST Client

---

## 참고

- [[../python-django|↑ Django area]]
- [Django 공식](https://docs.djangoproject.com/en/5.0/)
- [DRF 공식](https://www.django-rest-framework.org/)
- [drf-spectacular](https://drf-spectacular.readthedocs.io/)
- [simple-jwt](https://django-rest-framework-simplejwt.readthedocs.io/)
- [django-environ](https://django-environ.readthedocs.io/)
- [pytest-django](https://pytest-django.readthedocs.io/)
