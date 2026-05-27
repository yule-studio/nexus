---
title: "07. ViewSet 패턴 — mixin / filter / pagination / 권한"
kind: guide
status: current
created_at: 2026-05-26T12:05:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-django, guidelines, viewset, patterns]
---

# 07. ViewSet 패턴 — mixin / filter / pagination / 권한

**[[guidelines|↑ Hub]]** ← [[06-swagger-drf-spectacular]] · → 다음 [[08-monitoring]]

> DRF 의 ViewSet 변형 + filter / pagination / throttle 패턴.

---

## 1. ViewSet 종류

| 클래스 | 용도 |
| --- | --- |
| `ViewSet` | 메소드 직접 정의 (커스텀 액션) |
| `GenericViewSet` | `get_queryset` / `get_serializer` 기본 |
| `ReadOnlyModelViewSet` | list + retrieve 만 |
| `ModelViewSet` | 전체 CRUD |
| `mixins.X + GenericViewSet` | 필요한 mixin 만 |

---

## 2. Mixin 조합 — 필요한 액션만

```python
from rest_framework import mixins, viewsets


class MemoListCreateViewSet(
    mixins.ListModelMixin,
    mixins.CreateModelMixin,
    viewsets.GenericViewSet,                              # base
):
    """list + create 만."""
    queryset = Memo.objects.all()
    serializer_class = MemoSerializer
```

→ `DELETE` / `PATCH` 가 자동 노출 X.

---

## 3. Custom action — `@action`

```python
from rest_framework.decorators import action
from rest_framework.response import Response


class MemoViewSet(viewsets.ModelViewSet):
    ...

    @action(detail=True, methods=["post"], url_path="archive")
    def archive(self, request, pk=None):
        """POST /api/memos/{pk}/archive/"""
        memo = self.get_object()
        memo.is_archived = True
        memo.save(update_fields=["is_archived"])
        return Response({"ok": True, "data": MemoSerializer(memo).data, "error": None})

    @action(detail=False, methods=["get"], url_path="recent")
    def recent(self, request):
        """GET /api/memos/recent/"""
        qs = self.get_queryset()[:5]
        return Response(MemoSerializer(qs, many=True).data)
```

| `detail` | URL | 의미 |
| --- | --- | --- |
| `True` | `/api/memos/{pk}/<action>/` | 단건 액션 |
| `False` | `/api/memos/<action>/` | 컬렉션 액션 |

---

## 4. Filter — django-filter

```bash
pip install django-filter
```

`base.py`:
```python
INSTALLED_APPS += ["django_filters"]
REST_FRAMEWORK["DEFAULT_FILTER_BACKENDS"] = (
    "django_filters.rest_framework.DjangoFilterBackend",
    "rest_framework.filters.SearchFilter",
    "rest_framework.filters.OrderingFilter",
)
```

```python
# apps/memos/views.py
from django_filters import rest_framework as django_filters


class MemoFilter(django_filters.FilterSet):
    title = django_filters.CharFilter(lookup_expr="icontains")
    created_from = django_filters.DateTimeFilter(field_name="created_at", lookup_expr="gte")
    created_to = django_filters.DateTimeFilter(field_name="created_at", lookup_expr="lte")

    class Meta:
        model = Memo
        fields = ["title", "created_from", "created_to"]


class MemoViewSet(viewsets.ModelViewSet):
    filterset_class = MemoFilter
    search_fields = ["title", "content"]                  # SearchFilter — ?search=...
    ordering_fields = ["created_at", "updated_at"]        # ?ordering=-created_at
    ordering = ["-created_at"]                            # default
```

```bash
GET /api/memos/?title=hello&search=todo&ordering=-updated_at
```

---

## 5. Pagination — 종류별

### 5.1 PageNumberPagination (default)

```python
# settings/base.py
REST_FRAMEWORK = {
    ...,
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 20,
}
```

`?page=2&page_size=50` (page_size 활성화는 custom class 필요).

### 5.2 LimitOffsetPagination

```python
"DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.LimitOffsetPagination",
```

`?limit=50&offset=100`.

### 5.3 CursorPagination — 무한 스크롤 / 안정성

```python
# apps/memos/pagination.py
from rest_framework.pagination import CursorPagination


class MemoCursorPagination(CursorPagination):
    page_size = 20
    ordering = "-created_at"
    cursor_query_param = "cursor"


class MemoViewSet(viewsets.ModelViewSet):
    pagination_class = MemoCursorPagination
```

→ 큰 데이터 + 정확한 다음/이전 보장. SNS / 피드.

---

## 6. Permission — 세분화

```python
class MemoViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]

    def get_permissions(self):
        if self.action in ("update", "partial_update", "destroy"):
            return [IsAuthenticated(), IsOwner()]
        return super().get_permissions()
```

→ 액션별 권한 분기.

---

## 7. Throttle — rate limit

`base.py`:
```python
REST_FRAMEWORK = {
    ...,
    "DEFAULT_THROTTLE_CLASSES": (
        "rest_framework.throttling.AnonRateThrottle",
        "rest_framework.throttling.UserRateThrottle",
    ),
    "DEFAULT_THROTTLE_RATES": {
        "anon": "100/hour",
        "user": "1000/hour",
        "login": "5/minute",                              # scoped
    },
}
```

scoped throttle:
```python
from rest_framework.throttling import ScopedRateThrottle


class LoginView(TokenObtainPairView):
    throttle_classes = [ScopedRateThrottle]
    throttle_scope = "login"
```

→ 로그인은 분당 5번.

---

## 8. nested router — `drf-nested-routers`

```bash
pip install drf-nested-routers
```

```python
# apps/users/urls.py
from rest_framework_nested.routers import NestedDefaultRouter
from rest_framework.routers import DefaultRouter

from apps.accounts.views import UserViewSet
from apps.memos.views import UserMemoViewSet


router = DefaultRouter()
router.register("users", UserViewSet, basename="user")

user_router = NestedDefaultRouter(router, "users", lookup="user")
user_router.register("memos", UserMemoViewSet, basename="user-memos")

urlpatterns = router.urls + user_router.urls
```

생성:
```
GET    /api/users/{user_pk}/memos/
POST   /api/users/{user_pk}/memos/
GET    /api/users/{user_pk}/memos/{pk}/
```

→ `self.kwargs["user_pk"]` 로 접근.

---

## 9. select_related / prefetch_related 자동화

DRF 의 ViewSet 이 N+1 잘 생김. **`get_queryset` 의 의식적 최적화**:

```python
class MemoViewSet(viewsets.ModelViewSet):
    def get_queryset(self):
        qs = Memo.objects.select_related("user", "category").prefetch_related("tags")
        return qs.filter(user=self.request.user)
```

도구:
- `django-debug-toolbar` (dev) — SQL 시각화
- `nplusone` 패키지 — 자동 감지
- `silk` — request profiler

---

## 10. ViewSet 의 다양한 hook

| hook | 호출 시점 |
| --- | --- |
| `get_queryset(self)` | list / retrieve 의 baseline |
| `get_serializer_class(self)` | action 별 serializer 분기 |
| `get_object(self)` | retrieve / update / delete 의 단건 |
| `get_serializer_context(self)` | serializer 에 추가 context |
| `perform_create(self, serializer)` | save 직전 (user 주입 등) |
| `perform_update(self, serializer)` | update save 직전 |
| `perform_destroy(self, instance)` | delete 직전 (소프트 삭제 등) |
| `get_permissions(self)` | action 별 권한 |
| `get_throttles(self)` | action 별 throttle |

---

## 11. 자주 막히는 함정

### 함정 1 — `ModelViewSet` 모든 endpoint 노출
DELETE 같은 위험 endpoint 가 자동. **mixin 조합** 또는 명시.

### 함정 2 — `@action(url_path=...)` 누락
액션명이 그대로 URL → trailing slash 등 일관성 깨짐. 명시 권장.

### 함정 3 — `@action(detail=True/False)` 잘못
detail=True 면 `{pk}` 필요. URL 안 맞으면 404.

### 함정 4 — filterset_class vs filterset_fields
간단한 건 `filterset_fields = ["title"]`. 복잡한 lookup 은 FilterSet class.

### 함정 5 — CursorPagination 의 ordering 일관성
`ordering` 의 field 가 unique 안 함 → 같은 created_at 인 row 가 누락 / 중복. **`-created_at, -id`** 같은 tie-breaker.

### 함정 6 — Throttle 의 anon vs user
anon = IP 기반. 같은 NAT 뒤의 다중 사용자 = 잘못 차단 가능.

### 함정 7 — `get_permissions` 에서 instance return
**`[A(), B()]`** 처럼 instance 의 리스트. class 만 적으면 TypeError.

### 함정 8 — nested router 의 lookup field 충돌
부모 router 와 자식 router 의 `lookup` 이름 충돌 시 URL 못 풀음. 명시.

### 함정 9 — `perform_destroy` 가 hard delete
실수로 사라짐. 소프트 삭제 권장 (`is_deleted=True`).

### 함정 10 — `ordering` 가 unique 안 한 field
같은 정렬 키면 ORDER 비결정 → pagination 깨짐. 항상 id 같은 unique tie-breaker.

---

## 12. 다음

→ [[08-monitoring]] — django-prometheus + healthz
