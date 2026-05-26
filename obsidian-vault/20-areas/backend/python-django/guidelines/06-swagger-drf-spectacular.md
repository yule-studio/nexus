---
title: "06. Swagger — drf-spectacular OpenAPI"
kind: guide
status: current
created_at: 2026-05-26T11:45:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-django, guidelines, swagger, openapi, drf-spectacular]
---

# 06. Swagger — drf-spectacular OpenAPI

**[[guidelines|↑ Hub]]** ← [[05-auth-and-deployment]] · → 다음 [[07-viewset-patterns]]

> DRF 기본 schema generator 는 부족 → **drf-spectacular** (OpenAPI 3, Swagger UI / ReDoc + 강력한 introspection).

---

## 1. 설치

```bash
pip install drf-spectacular
```

`base.py`:
```python
INSTALLED_APPS += ["drf_spectacular"]

REST_FRAMEWORK = {
    ...,
    "DEFAULT_SCHEMA_CLASS": "drf_spectacular.openapi.AutoSchema",
}

SPECTACULAR_SETTINGS = {
    "TITLE": "Memo API",
    "DESCRIPTION": "간단한 메모 CRUD + 회원가입 / JWT",
    "VERSION": "1.0.0",
    "SERVE_INCLUDE_SCHEMA": False,
    "SCHEMA_PATH_PREFIX": r"/api/",
    "COMPONENT_SPLIT_REQUEST": True,                       # Request / Response schema 분리
    "SWAGGER_UI_SETTINGS": {
        "persistAuthorization": True,                       # 새로고침에도 토큰 유지
        "displayRequestDuration": True,
    },
    "SERVERS": [
        {"url": "http://localhost:8000", "description": "local"},
        {"url": "https://api.example.com", "description": "prod"},
    ],
    "TAGS": [
        {"name": "auth", "description": "회원가입 / 로그인"},
        {"name": "memos", "description": "메모 CRUD"},
    ],
    "SECURITY": [{"BearerAuth": []}],
}
```

---

## 2. URL 등록 — 자동 generation

```python
# config/urls.py
from django.urls import path, include
from drf_spectacular.views import (
    SpectacularAPIView,
    SpectacularSwaggerView,
    SpectacularRedocView,
)


urlpatterns = [
    path("admin/", admin.site.urls),
    path("api/auth/", include("apps.accounts.urls")),
    path("api/memos/", include("apps.memos.urls")),

    # OpenAPI schema
    path("api/schema/", SpectacularAPIView.as_view(), name="schema"),
    # Swagger UI
    path("api/docs/", SpectacularSwaggerView.as_view(url_name="schema"), name="docs"),
    # ReDoc
    path("api/redoc/", SpectacularRedocView.as_view(url_name="schema"), name="redoc"),
]
```

- **Swagger UI**: <http://localhost:8000/api/docs/>
- **ReDoc**: <http://localhost:8000/api/redoc/>
- **OpenAPI JSON / YAML**: <http://localhost:8000/api/schema/>

---

## 3. ViewSet / endpoint metadata

```python
# apps/memos/views.py
from drf_spectacular.utils import extend_schema, extend_schema_view, OpenApiResponse, OpenApiParameter

from .serializers import MemoSerializer, MemoCreateSerializer


@extend_schema_view(
    list=extend_schema(
        summary="내 메모 목록",
        description="현재 로그인 사용자의 메모를 페이지 단위 조회",
        tags=["memos"],
        parameters=[
            OpenApiParameter("page", int, OpenApiParameter.QUERY, description="페이지 번호"),
            OpenApiParameter("page_size", int, OpenApiParameter.QUERY, description="페이지 크기"),
        ],
    ),
    create=extend_schema(
        summary="메모 생성",
        request=MemoCreateSerializer,
        responses={201: MemoSerializer, 400: OpenApiResponse(description="validation 실패")},
        tags=["memos"],
    ),
    retrieve=extend_schema(summary="메모 단건 조회", tags=["memos"]),
    partial_update=extend_schema(summary="메모 부분 수정", tags=["memos"]),
    destroy=extend_schema(summary="메모 삭제", tags=["memos"]),
)
class MemoViewSet(viewsets.ModelViewSet):
    ...
```

→ Swagger UI 에 모든 endpoint + 설명 + parameter / request / response schema 자동.

---

## 4. APIView — `@extend_schema` 데코레이터

```python
# apps/accounts/views.py
from drf_spectacular.utils import extend_schema, OpenApiExample


@extend_schema(
    summary="회원가입",
    request=SignupSerializer,
    responses={
        201: OpenApiResponse(description="가입 성공"),
        400: OpenApiResponse(description="중복 / validation 실패"),
    },
    tags=["auth"],
    examples=[
        OpenApiExample(
            "정상 가입",
            value={"email": "yuchan@example.com", "password": "P@ssw0rd1!", "name": "yuchan"},
            request_only=True,
        ),
    ],
)
class SignupView(APIView):
    ...
```

---

## 5. JWT Bearer — Swagger UI Authorize

`SPECTACULAR_SETTINGS` 에 이미 `"SECURITY": [{"BearerAuth": []}]`. simple-jwt 의 `JWTAuthentication` 이 자동 등록 → Swagger UI 우측 상단 `Authorize` 버튼.

수동으로도:

```python
SPECTACULAR_SETTINGS["SECURITY_SCHEMES"] = {
    "BearerAuth": {
        "type": "http",
        "scheme": "bearer",
        "bearerFormat": "JWT",
    },
}
```

> Authorize 클릭 → `<JWT>` (Bearer 빼고 토큰만) → 이후 모든 요청 자동 Authorization 헤더.

---

## 6. Serializer 의 OpenAPI 메타

```python
from drf_spectacular.utils import extend_schema_field


class MemoSerializer(serializers.ModelSerializer):
    class Meta:
        model = Memo
        fields = ["id", "title", "content", "created_at", "updated_at"]

    @extend_schema_field(serializers.IntegerField())
    def get_word_count(self, obj) -> int:
        return len(obj.content.split())
```

또는 `Serializer` 의 `help_text`:

```python
class SignupSerializer(serializers.Serializer):
    email = serializers.EmailField(help_text="유일한 이메일 — 로그인 ID 로 사용")
    password = serializers.CharField(
        min_length=8, write_only=True,
        help_text="8자 이상, 숫자 / 기호 권장",
    )
    name = serializers.CharField(max_length=50, help_text="표시 이름")
```

→ Swagger 의 schema field description.

---

## 7. schema 생성 (CI 검증)

```bash
python manage.py spectacular --file openapi.yaml
```

→ static file 로 export. CI 에서 diff 검증 권장:

```bash
# CI step
python manage.py spectacular --file openapi.yaml.new
diff openapi.yaml openapi.yaml.new || exit 1
```

→ API 변경 = openapi.yaml 도 commit. front / mobile 팀이 의존.

---

## 8. 운영 — Swagger 비활성

운영 노출 = API 구조 / endpoint 공개. **운영 비활성** 또는 admin only:

```python
# config/urls.py
from django.conf import settings

urlpatterns = [
    ...,
]

if settings.DEBUG:
    urlpatterns += [
        path("api/schema/", SpectacularAPIView.as_view()),
        path("api/docs/", SpectacularSwaggerView.as_view(url_name="schema")),
    ]
```

또는 정확한 권한:

```python
class StaffOnlySwaggerView(SpectacularSwaggerView):
    permission_classes = [IsAdminUser]
```

---

## 9. Tag 기반 grouping

```python
SPECTACULAR_SETTINGS["TAGS"] = [
    {"name": "auth", "description": "회원가입 / 로그인 / 토큰"},
    {"name": "memos", "description": "메모 CRUD"},
    {"name": "admin", "description": "관리자"},
]
```

`@extend_schema(tags=["auth"])` 로 endpoint 마다 tag 지정 → Swagger UI 가 tag 별 group.

---

## 10. 자주 막히는 함정

### 함정 1 — `DEFAULT_SCHEMA_CLASS` 설정 X
spectacular 가 적용 X. 누락하면 DRF 기본 (부정확).

### 함정 2 — `JWTAuthentication` 의 Bearer 인식 X
spectacular 가 simple-jwt 자동 인식. 만약 인식 X 면 `SPECTACULAR_SETTINGS["SECURITY_SCHEMES"]` 수동.

### 함정 3 — `@extend_schema(responses=...)` 누락
응답 schema 가 부정확. 명시 권장.

### 함정 4 — `SERVE_INCLUDE_SCHEMA=True` (default)
모든 endpoint 에 `/schema/` 도 함께 노출. 보통 False 권장.

### 함정 5 — Serializer 의 `SerializerMethodField` schema 부정확
`@extend_schema_field` 로 명시 권장.

### 함정 6 — 운영 노출
endpoint 구조 / serializer 전체 공개. **운영 비활성**.

### 함정 7 — `COMPONENT_SPLIT_REQUEST=False`
request / response 의 schema 가 한 component → readonly field 도 request 에 노출. **True 권장**.

### 함정 8 — schema 검증 CI 없음
silent 변경 → front 가 깨짐. CI 에서 `spectacular --file` diff.

### 함정 9 — `persistAuthorization=False`
Swagger UI 새로고침 시 토큰 사라짐. True 가 dev 편함.

### 함정 10 — `@extend_schema` 위치 — class 위 vs method 위
ViewSet 의 method 별 메타는 **`@extend_schema_view`** 로 통합. APIView 의 단일 method 는 직접 `@extend_schema`.

---

## 11. 다음

→ [[07-viewset-patterns]] — ModelViewSet / mixin / 권한 / filter / pagination
