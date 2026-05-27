---
title: "08. Monitoring — django-prometheus + healthz + Sentry"
kind: guide
status: current
created_at: 2026-05-26T12:25:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-django, guidelines, monitoring, prometheus, healthz]
---

# 08. Monitoring — django-prometheus + healthz + Sentry

**[[guidelines|↑ Hub]]** ← [[07-viewset-patterns]] · → 다음 [[09-logging-mdc]]

---

## 1. healthz / readyz endpoint

```python
# apps/common/views.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.permissions import AllowAny
from django.db import connections
from django.db.utils import OperationalError


class HealthzView(APIView):
    """가벼운 liveness."""
    permission_classes = [AllowAny]
    authentication_classes = []

    def get(self, request):
        return Response({"status": "ok"})


class ReadyzView(APIView):
    """readiness — DB ping 포함."""
    permission_classes = [AllowAny]
    authentication_classes = []

    def get(self, request):
        components = {}
        ok = True
        try:
            connections["default"].cursor().execute("SELECT 1")
            components["db"] = "up"
        except OperationalError as e:
            components["db"] = f"down: {e}"
            ok = False
        return Response(
            {"status": "ready" if ok else "not-ready", "components": components},
            status=200 if ok else 503,
        )
```

```python
# config/urls.py
from apps.common.views import HealthzView, ReadyzView


urlpatterns = [
    path("healthz/", HealthzView.as_view()),
    path("readyz/", ReadyzView.as_view()),
    ...
]
```

> healthz / readyz 는 **인증 미적용**. k8s probe 가 익명 호출.

---

## 2. Kubernetes probe

```yaml
livenessProbe:
  httpGet:
    path: /healthz/
    port: 8000
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /readyz/
    port: 8000
  initialDelaySeconds: 5
  periodSeconds: 5
```

---

## 3. Prometheus — django-prometheus

```bash
pip install django-prometheus
```

`base.py`:
```python
INSTALLED_APPS += ["django_prometheus"]

MIDDLEWARE = [
    "django_prometheus.middleware.PrometheusBeforeMiddleware",     # 가장 위
    "django.middleware.security.SecurityMiddleware",
    ...
    "django_prometheus.middleware.PrometheusAfterMiddleware",      # 가장 아래
]
```

```python
# config/urls.py
urlpatterns = [
    ...,
    path("", include("django_prometheus.urls")),                   # /metrics
]
```

→ `/metrics` 자동.

```bash
curl http://localhost:8000/metrics
# django_http_requests_total_by_method{method="GET"} 142
# django_http_requests_latency_seconds_by_view_method_bucket{view="apps.memos.views.MemoViewSet",method="GET",le="0.5"} 100
# django_db_execute_total_by_database{alias="default"} 320
```

---

## 4. DB / Cache metric — 자동

`base.py`:
```python
DATABASES = {
    "default": {
        **env.db("DATABASE_URL"),
        "ENGINE": "django_prometheus.db.backends.mysql",            # 또는 postgresql
    },
}

CACHES = {
    "default": {
        "BACKEND": "django_prometheus.cache.backends.redis.RedisCache",
        "LOCATION": env("REDIS_URL"),
    },
}
```

→ DB query / cache hit / miss metric 자동.

---

## 5. Custom metric — prometheus_client

```python
# apps/memos/metrics.py
from prometheus_client import Counter, Histogram


memos_created = Counter("memos_created_total", "메모 생성 횟수", ["env"])
slow_operation = Histogram("slow_operation_seconds", "느린 작업 시간")
```

```python
# views.py
from apps.memos.metrics import memos_created


class MemoViewSet(viewsets.ModelViewSet):
    def perform_create(self, serializer):
        serializer.save(user=self.request.user)
        memos_created.labels(env=settings.ENV).inc()
```

→ `/metrics` 자동 노출.

---

## 6. Prometheus config

```yaml
# prometheus.yml
scrape_configs:
  - job_name: django-memo
    metrics_path: /metrics
    static_configs:
      - targets: ["memo-api:8000"]
    scrape_interval: 15s
```

---

## 7. Sentry 통합

```bash
pip install sentry-sdk
```

```python
# config/settings/prod.py
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration


sentry_sdk.init(
    dsn=env("SENTRY_DSN"),
    integrations=[DjangoIntegration()],
    traces_sample_rate=0.1,                                # 10% APM
    environment=env("ENV", default="prod"),
    release="memo-api@1.0.0",
    send_default_pii=False,                                # PII 안 보냄
)
```

→ uncaught exception + 4xx 일부 자동 수집.

### 7.1 manual context

```python
from sentry_sdk import set_user, capture_exception


def get_current_user(request):
    user = ...
    set_user({"id": user.id, "email": user.email})
    return user


try:
    risky_operation()
except Exception as e:
    capture_exception(e)
    raise
```

---

## 8. /metrics 보호

`/metrics` 외부 노출 = endpoint URL / latency 정보 공개. **내부 망 only**:

```python
# apps/common/middleware.py
from django.http import JsonResponse


class RestrictMetricsMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        if request.path == "/metrics":
            client_ip = request.META.get("HTTP_X_FORWARDED_FOR", "").split(",")[0].strip()
            if not client_ip:
                client_ip = request.META.get("REMOTE_ADDR", "")
            if not (client_ip.startswith("10.") or client_ip.startswith("172.")):
                return JsonResponse({"detail": "Forbidden"}, status=403)
        return self.get_response(request)
```

→ Nginx 단에서 `/metrics` 차단도 권장.

---

## 9. Grafana 추천 대시보드

- **Django Observability** — django-prometheus 의 메트릭 시각화
- **Django Monitoring** ([grafana.com/dashboards](https://grafana.com/grafana/dashboards/))

주요 panel:
- request rate / latency (p50 / p95 / p99)
- error rate (5xx / 4xx)
- DB query rate / latency
- cache hit ratio
- worker memory (RSS)

---

## 10. 자주 막히는 함정

### 함정 1 — `/metrics` 외부 노출
endpoint URL / latency 노출. **내부 망 only**.

### 함정 2 — django-prometheus middleware 위치
`PrometheusBeforeMiddleware` = 가장 위, `PrometheusAfterMiddleware` = 가장 아래. 다른 위치 = 부정확한 시간.

### 함정 3 — DB engine 변경 후 migration 재실행
`django_prometheus.db.backends.mysql` 로 변경 시 schema 변동 X. 안전하지만 표시.

### 함정 4 — Sentry traces_sample_rate=1.0
모든 요청 trace → 무거움 + 비용. prod 0.01 ~ 0.1.

### 함정 5 — multi-worker (gunicorn `--workers 4`)
worker 별 in-memory counter → 별도 시계열. `prometheus_client` 의 multiprocess mode + `PROMETHEUS_MULTIPROC_DIR`.

### 함정 6 — `readyz` 가 무거움 (외부 의존 다 ping)
slow → probe fail. 가벼운 query 만.

### 함정 7 — healthz 에 인증 적용
k8s probe = 401 → restart 폭주. **`permission_classes = [AllowAny]`** + `authentication_classes = []`.

### 함정 8 — Sentry 의 4xx 알람
`AuthenticationFailed` 같은 4xx 도 Sentry → 알람 폭주. `before_send` filter 또는 5xx only.

### 함정 9 — `/healthz` vs `/healthz/`
Django 의 `APPEND_SLASH=True` (default) 면 자동 redirect. k8s probe path 와 일치 확인.

### 함정 10 — Sentry 의 PII (`send_default_pii=True`)
요청 body / user info 자동 전송. GDPR 위험. **False** 권장.

---

## 11. 다음

→ [[09-logging-mdc]] — django-log-request-id + JSON log
