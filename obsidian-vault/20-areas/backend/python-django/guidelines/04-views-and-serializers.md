---
title: "04. DRF ViewSet + Serializer + Router + Exception"
kind: guide
status: current
created_at: 2026-05-26T11:05:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-django, guidelines, drf, viewset, serializer]
---

# 04. DRF ViewSet + Serializer + Router + Exception

**[[guidelines|↑ Hub]]** ← [[03-models-and-orm]] · → 다음 [[05-auth-and-deployment]]

> DRF 의 핵심 = **Serializer** (validation + 직렬화) + **ViewSet** (CRUD 자동) + **Router** (URL 자동) + **Exception handler** (envelope 통일).

---

## 1. Serializer — pydantic / marshmallow 대응

```python
# apps/memos/serializers.py
from rest_framework import serializers

from .models import Memo


class MemoSerializer(serializers.ModelSerializer):
    class Meta:
        model = Memo
        fields = ["id", "title", "content", "created_at", "updated_at"]
        read_only_fields = ["id", "created_at", "updated_at"]


class MemoCreateSerializer(serializers.Serializer):
    title = serializers.CharField(max_length=200, min_length=1)
    content = serializers.CharField(max_length=10_000, allow_blank=True, default="")


class MemoUpdateSerializer(serializers.Serializer):
    title = serializers.CharField(max_length=200, min_length=1, required=False)
    content = serializers.CharField(max_length=10_000, allow_blank=True, required=False)
```

### 1.1 ModelSerializer vs Serializer

| 용도 | 종류 |
| --- | --- |
| Model → JSON (응답) | `ModelSerializer` — 자동 매핑 |
| 입력 validation (Model 과 다른 shape) | `Serializer` — 명시적 field |

> 본 가이드는 **응답 = ModelSerializer / 입력 = Serializer** 로 분리. 입력과 출력의 shape 차이를 명시.

---

## 2. ViewSet — CRUD 자동

```python
# apps/memos/views.py
from rest_framework import viewsets, status
from rest_framework.response import Response

from .models import Memo
from .serializers import MemoSerializer, MemoCreateSerializer, MemoUpdateSerializer


class MemoViewSet(viewsets.ModelViewSet):
    serializer_class = MemoSerializer
    # permission_classes = [IsAuthenticated]              # base settings 에서 적용

    def get_queryset(self):
        """현재 로그인 사용자의 메모만."""
        return (
            Memo.objects.select_related("user")
            .filter(user=self.request.user)
            .order_by("-created_at")
        )

    def perform_create(self, serializer):
        """user 자동 주입."""
        serializer.save(user=self.request.user)

    def get_serializer_class(self):
        if self.action == "create":
            return MemoCreateSerializer
        if self.action in ("update", "partial_update"):
            return MemoUpdateSerializer
        return MemoSerializer
```

`ModelViewSet` 이 자동으로 제공:
- `list` (`GET /`)
- `create` (`POST /`)
- `retrieve` (`GET /{id}/`)
- `update` (`PUT /{id}/`)
- `partial_update` (`PATCH /{id}/`)
- `destroy` (`DELETE /{id}/`)

---

## 3. Router + URL

```python
# apps/memos/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter

from .views import MemoViewSet


router = DefaultRouter()
router.register("", MemoViewSet, basename="memo")

urlpatterns = [
    path("", include(router.urls)),
]
```

```python
# config/urls.py
urlpatterns = [
    path("admin/", admin.site.urls),
    path("api/auth/", include("apps.accounts.urls")),
    path("api/memos/", include("apps.memos.urls")),
]
```

생성된 endpoint:
```
GET    /api/memos/         → list
POST   /api/memos/         → create
GET    /api/memos/{id}/    → retrieve
PUT    /api/memos/{id}/    → update
PATCH  /api/memos/{id}/    → partial_update
DELETE /api/memos/{id}/    → destroy
```

---

## 4. APIView — ViewSet 안 맞을 때

```python
# apps/accounts/views.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

from .serializers import SignupSerializer
from .models import User


class SignupView(APIView):
    permission_classes = []                                # 공개

    def post(self, request):
        serializer = SignupSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        user = User.objects.create_user(**serializer.validated_data)
        return Response(
            {"ok": True, "data": {"id": user.id, "email": user.email}, "error": None},
            status=status.HTTP_201_CREATED,
        )
```

---

## 5. Serializer 의 validation

### 5.1 field-level

```python
class SignupSerializer(serializers.Serializer):
    email = serializers.EmailField()
    password = serializers.CharField(min_length=8, write_only=True)
    name = serializers.CharField(max_length=50)

    def validate_email(self, value: str) -> str:
        if User.objects.filter(email=value).exists():
            raise serializers.ValidationError("이미 가입된 이메일")
        return value

    def validate_password(self, value: str) -> str:
        if value.isalpha():
            raise serializers.ValidationError("숫자 / 기호 포함 필요")
        return value
```

### 5.2 object-level (여러 field 관계)

```python
class TransferSerializer(serializers.Serializer):
    from_account = serializers.IntegerField()
    to_account = serializers.IntegerField()
    amount = serializers.IntegerField(min_value=1)

    def validate(self, attrs):
        if attrs["from_account"] == attrs["to_account"]:
            raise serializers.ValidationError("같은 계좌 송금 X")
        return attrs
```

### 5.3 validation 실패 = 자동 400

```
POST /api/auth/signup/
{"email": "bad", "password": "1234567", "name": ""}

→ HTTP 400
{
  "email": ["Enter a valid email address."],
  "password": ["Ensure this field has at least 8 characters."],
  "name": ["This field may not be blank."]
}
```

→ 본 가이드의 envelope 통일은 [[#7 envelope|§7]] 의 exception handler.

---

## 6. Pagination

`base.py`:
```python
REST_FRAMEWORK = {
    "DEFAULT_PAGINATION_CLASS":
        "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 20,
}
```

```bash
GET /api/memos/?page=2
```

응답:
```json
{
    "count": 100,
    "next": "http://localhost:8000/api/memos/?page=3",
    "previous": "http://localhost:8000/api/memos/?page=1",
    "results": [...]
}
```

> `LimitOffsetPagination` / `CursorPagination` 도 있음 — [[07-viewset-patterns#5 pagination|07 §5]].

---

## 7. envelope 통일 — exception handler

DRF 의 default 응답 shape 는 endpoint 마다 다름. 통일 envelope:

```json
{ "ok": true, "data": {...}, "error": null, "request_id": "..." }
{ "ok": false, "data": null, "error": {"code": "VALIDATION", "message": "..."}, "request_id": "..." }
```

### 7.1 custom exception handler

```python
# config/exception_handlers.py
from rest_framework.views import exception_handler
from rest_framework.response import Response


def envelope_exception_handler(exc, context):
    """모든 예외를 envelope 로 변환."""
    response = exception_handler(exc, context)

    if response is None:
        # 미처리 예외 — 500
        return Response(
            {
                "ok": False,
                "data": None,
                "error": {"code": "INTERNAL", "message": str(exc)},
            },
            status=500,
        )

    # DRF 가 처리한 4xx — envelope 변환
    data = response.data
    if isinstance(data, dict) and "detail" in data:
        err_msg = str(data["detail"])
        code = data.get("code", "ERROR")
    else:
        err_msg = "validation failed"
        code = "VALIDATION"

    response.data = {
        "ok": False,
        "data": None,
        "error": {"code": code, "message": err_msg, "fields": data if response.status_code == 400 else None},
    }
    return response
```

`base.py`:
```python
REST_FRAMEWORK = {
    ...,
    "EXCEPTION_HANDLER": "config.exception_handlers.envelope_exception_handler",
}
```

### 7.2 custom exception

```python
# apps/common/exceptions.py
from rest_framework.exceptions import APIException


class NotFound(APIException):
    status_code = 404
    default_detail = "리소스를 찾을 수 없음"
    default_code = "NOT_FOUND"


class Forbidden(APIException):
    status_code = 403
    default_detail = "권한 없음"
    default_code = "FORBIDDEN"


class Conflict(APIException):
    status_code = 409
    default_detail = "충돌"
    default_code = "CONFLICT"
```

사용:
```python
def get_object(self):
    obj = self.get_queryset().filter(pk=self.kwargs["pk"]).first()
    if obj is None:
        raise NotFound("메모를 찾을 수 없음")
    if obj.user != self.request.user:
        raise Forbidden("본인 메모만 접근 가능")
    return obj
```

---

## 8. envelope 성공 응답 — Renderer (선택)

매 응답마다 envelope wrapping = 번거로움. **custom Renderer**:

```python
# config/renderers.py
from rest_framework.renderers import JSONRenderer


class EnvelopeJSONRenderer(JSONRenderer):
    def render(self, data, accepted_media_type=None, renderer_context=None):
        response = renderer_context.get("response") if renderer_context else None
        is_error = response is not None and response.status_code >= 400

        if is_error:
            # exception_handler 가 이미 envelope 변환
            wrapped = data
        else:
            wrapped = {"ok": True, "data": data, "error": None}

        return super().render(wrapped, accepted_media_type, renderer_context)
```

```python
# settings/base.py
REST_FRAMEWORK = {
    ...,
    "DEFAULT_RENDERER_CLASSES": [
        "config.renderers.EnvelopeJSONRenderer",
        "rest_framework.renderers.BrowsableAPIRenderer",     # dev 만 권장
    ],
}
```

→ 모든 성공 응답이 자동 envelope.

---

## 9. PATCH (partial_update) 의 patch 처리

```python
def partial_update(self, request, *args, **kwargs):
    instance = self.get_object()
    serializer = self.get_serializer(instance, data=request.data, partial=True)
    serializer.is_valid(raise_exception=True)
    self.perform_update(serializer)
    return Response(MemoSerializer(instance).data)
```

> `partial=True` = 명시된 field 만 update (다른 field 미변경).

---

## 10. 자주 막히는 함정

### 함정 1 — `ModelSerializer` 의 `fields = "__all__"`
미래 추가 field 가 자동 노출 → 민감 정보 유출 위험. **명시 권장**.

### 함정 2 — `ModelViewSet` 가 모든 endpoint 노출
DELETE 같은 위험한 것도 자동. 필요 endpoint 만 = `ReadOnlyModelViewSet` 또는 mixin 조합.

### 함정 3 — `get_queryset` 에서 user filtering 누락
다른 사람의 데이터 노출. **항상 `filter(user=self.request.user)`**.

### 함정 4 — `perform_create` 안 쓰고 직접 `save()`
user 주입 빠짐. ModelViewSet 의 hook 사용.

### 함정 5 — Serializer 의 `password` 가 read
응답에 password hash 노출. **`write_only=True`** 필수.

### 함정 6 — `is_valid()` 후 `validated_data` 사용 안 함
`is_valid()` 호출 안 하면 `validated_data` 비어있음. 반드시 호출.

### 함정 7 — Router `basename` 누락
`queryset` 명시 안 한 ViewSet 은 `basename` 필수. 누락 시 `ImproperlyConfigured`.

### 함정 8 — Browsable API 운영
endpoint 구조 + serializer 노출. **prod 에선 JSONRenderer 만**.

### 함정 9 — exception 안 잡힘 (response None)
`Http404` 같은 Django 예외도 DRF exception_handler 가 처리. 미처리 예외만 500.

### 함정 10 — Serializer 의 `validate_<field>` 와 default 충돌
default = field 가 입력 안 됨 → `validate_<field>` 호출 안 됨. 의도 구분.

### 함정 11 — `PUT` vs `PATCH` 혼용
PUT = 전체 교체 (required field 모두 필요). PATCH = 부분. ModelViewSet 이 둘 다 자동.

### 함정 12 — N+1 in serializer
ManyRelatedField 마다 SQL. **`get_queryset` 에서 `select_related/prefetch_related`**.

---

## 11. 다음

→ [[05-auth-and-deployment]] — simple-jwt + 권한 + gunicorn + Nginx + Docker
