---
title: "08. Monitoring — prometheus-fastapi-instrumentator + healthz"
kind: guide
status: current
created_at: 2026-05-27T16:15:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, guidelines, monitoring, prometheus]
---

# 08. Monitoring — prometheus-fastapi-instrumentator + healthz

**[[guidelines|↑ Hub]]** ← [[07-controller-patterns]] · → 다음 [[09-logging-mdc]]

---

## 1. healthz endpoint

```python
# app/api/health.py
from fastapi import APIRouter, status
from sqlalchemy import text

from app.deps import SessionDep

router = APIRouter()


@router.get("/healthz", status_code=status.HTTP_200_OK, tags=["health"])
async def healthz():
    """가벼운 liveness."""
    return {"status": "ok"}


@router.get("/readyz", tags=["health"])
async def readyz(db: SessionDep):
    """readiness — DB 등 의존 포함."""
    try:
        await db.execute(text("SELECT 1"))
        return {"status": "ready", "components": {"db": "up"}}
    except Exception as e:
        return JSONResponse(
            status_code=503,
            content={"status": "not-ready", "components": {"db": str(e)}},
        )
```

```python
# main.py
app.include_router(health.router)
```

---

## 2. Kubernetes probe

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8000
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /readyz
    port: 8000
  initialDelaySeconds: 5
  periodSeconds: 5
```

---

## 3. Prometheus metrics — prometheus-fastapi-instrumentator

```bash
pip install prometheus-fastapi-instrumentator
```

```python
# app/main.py
from prometheus_fastapi_instrumentator import Instrumentator


def create_app() -> FastAPI:
    app = FastAPI(...)

    # ...

    # instrumentator
    instrumentator = Instrumentator(
        should_group_status_codes=True,
        should_ignore_untemplated=True,                # 정적 URL 무시
        excluded_handlers=["/healthz", "/readyz", "/metrics"],
        env_var_name="ENABLE_METRICS",                  # env 로 on/off
    )
    instrumentator.instrument(app)
    instrumentator.expose(app, endpoint="/metrics", include_in_schema=False)

    return app
```

`/metrics` 자동:

```bash
curl http://localhost:8000/metrics
# http_request_duration_seconds_bucket{method="GET",status="2xx",handler="/api/v1/memos",le="0.5"} 142
# http_request_size_bytes_sum{...} ...
# http_response_size_bytes_count{...} ...
```

---

## 4. Prometheus config

```yaml
# prometheus.yml
scrape_configs:
  - job_name: fastapi-memo
    metrics_path: /metrics
    static_configs:
      - targets: ["memo-api:8000"]
    scrape_interval: 15s
```

---

## 5. 자주 보는 메트릭

| metric | 의미 |
| --- | --- |
| `http_requests_total{method, status, handler}` | 요청 수 |
| `http_request_duration_seconds_*` | 응답 시간 (histogram) |
| `http_request_size_bytes_*` | request body 크기 |
| `http_response_size_bytes_*` | response body 크기 |
| `process_cpu_seconds_total` | CPU 사용 |
| `process_resident_memory_bytes` | 메모리 |

---

## 6. Custom metric — prometheus_client

```python
from prometheus_client import Counter, Histogram

memos_created = Counter("memos_created_total", "메모 생성 횟수", ["env"])
slow_operation_time = Histogram("slow_operation_seconds", "느린 작업 시간")


@router.post("/memos")
async def create_memo(...):
    memo = await memo_service.create_memo(...)
    memos_created.labels(env="prod").inc()
    return ...


# context manager
@router.get("/slow")
async def slow():
    with slow_operation_time.time():
        await heavy_operation()
    return {"done": True}
```

→ `/metrics` 에 자동 노출.

---

## 7. Sentry 통합

```bash
pip install sentry-sdk[fastapi]
```

```python
# app/main.py
import sentry_sdk
from sentry_sdk.integrations.fastapi import FastApiIntegration
from sentry_sdk.integrations.starlette import StarletteIntegration


sentry_sdk.init(
    dsn=settings.SENTRY_DSN,
    integrations=[
        FastApiIntegration(),
        StarletteIntegration(),
    ],
    traces_sample_rate=0.1,
    environment=settings.ENV,
    release="memo-api@1.0.0",
)
```

→ uncaught exception 자동. `traces_sample_rate=0.1` 이면 10% APM trace.

---

## 8. 운영 — /metrics 보호

`/metrics` 외부 노출 = endpoint URL 공개 + APM 우회. **내부 망 only**:

```python
@app.middleware("http")
async def restrict_metrics(request: Request, call_next):
    if request.url.path == "/metrics":
        client_ip = request.headers.get("X-Forwarded-For", "").split(",")[0].strip() \
                    or request.client.host
        if not client_ip.startswith("10.") and not client_ip.startswith("172."):
            return JSONResponse(status_code=403, content={"detail": "Forbidden"})
    return await call_next(request)
```

또는 Nginx 단에서 `/metrics` 차단.

---

## 9. Grafana 추천 대시보드

- **FastAPI Observability** — Instrumentator 의 모든 메트릭
- **ASGI / Starlette Dashboard**

---

## 10. 자주 막히는 함정

### 함정 1 — `/metrics` 외부 노출
endpoint URL / latency 노출. **내부 망 only**.

### 함정 2 — `should_ignore_untemplated=False`
`/items/123` 처럼 동적 URL 마다 별도 시계열 → cardinality 폭발. **True 권장**.

### 함정 3 — Sentry 의 4xx alert
`BusinessException` 4xx 도 Sentry → 알람 폭주. `before_send` filter 또는 5xx 만.

### 함정 4 — multi-worker (uvicorn `--workers 4`) + in-memory metric
worker 별 별도 시계열. **`prometheus_client.multiprocess`** 모드 또는 별도 exporter.

### 함정 5 — `readyz` 가 무거움
k8s probe fail. **가벼운 ping** (SELECT 1).

### 함정 6 — async middleware + sync metric 호출
`Counter.inc()` 는 sync OK (thread-safe). 단 무거운 metric 호출은 event loop block 위험.

### 함정 7 — Sentry traces_sample_rate=1.0
모든 요청 trace → 부담. dev 0.1, prod 0.01 ~ 0.1.

### 함정 8 — `Instrumentator` 의 endpoint 가 `response_model` 적용
`/metrics` 가 OpenAPI 에 노출 — `include_in_schema=False` 권장.

---

## 11. 다음

✅ Monitoring 끝.

→ [[09-logging-mdc]] — structlog + asgi-correlation-id
