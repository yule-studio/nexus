---
title: "08. Monitoring — prometheus-flask-exporter + healthz"
kind: guide
status: current
created_at: 2026-05-27T13:45:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-flask, guidelines, monitoring, prometheus]
---

# 08. Monitoring — prometheus-flask-exporter + healthz

**[[guidelines|↑ Hub]]** ← [[07-controller-patterns]] · → 다음 [[09-logging-mdc]]

> Spring Boot Actuator 동등 — Flask 진영은 별도 라이브러리 (prometheus-flask-exporter / Flask-Healthz / 직접).

---

## 1. healthz endpoint — 직접

```python
# app/api/health.py
from flask import Blueprint
from sqlalchemy import text

from app.extensions import db

health_bp = Blueprint("health", __name__)


@health_bp.get("/healthz")
def healthz():
    """가벼운 liveness — process 살았는지."""
    return {"status": "ok"}, 200


@health_bp.get("/readyz")
def readyz():
    """readiness — 트래픽 받을 준비? DB / Redis 등 외부 의존 포함."""
    try:
        db.session.execute(text("SELECT 1"))
        return {"status": "ready", "components": {"db": "up"}}, 200
    except Exception as e:
        return {"status": "not-ready", "components": {"db": str(e)}}, 503
```

```python
# create_app 안
app.register_blueprint(health_bp)
```

→ 외부 노출 (LB / k8s probe) 위해 `@jwt_required()` 없어야.

---

## 2. Kubernetes probe

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 5000
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /readyz
    port: 5000
  initialDelaySeconds: 5
  periodSeconds: 5
```

---

## 3. Flask-Healthz (선택)

`Flask-Healthz` = healthz 자동:

```bash
pip install Flask-Healthz
```

```python
# app/__init__.py
from flask_healthz import healthz, HealthError

app.register_blueprint(healthz, url_prefix="/healthz")

# 헬스 함수 등록
def liveness():
    pass                                              # 예외 안 던지면 OK

def readiness():
    try:
        db.session.execute(text("SELECT 1"))
    except Exception as e:
        raise HealthError(f"DB: {e}")

app.config["HEALTHZ"] = {
    "live": "app:liveness",
    "ready": "app:readiness",
}
```

→ `/healthz/live`, `/healthz/ready` 자동 생성.

---

## 4. Prometheus metrics — prometheus-flask-exporter

```bash
pip install prometheus-flask-exporter
```

```python
# app/__init__.py
from prometheus_flask_exporter import PrometheusMetrics


def create_app(...) -> Flask:
    app = Flask(__name__)
    # ...

    metrics = PrometheusMetrics(
        app,
        defaults_prefix="memo_api",
        group_by="endpoint",                          # endpoint 별 grouping
    )

    # static info
    metrics.info("app_info", "Application info",
                 version="1.0.0", env=app.config.get("ENV", "prod"))

    return app
```

`/metrics` endpoint 자동 노출:

```bash
curl http://localhost:5000/metrics
# # HELP memo_api_flask_http_request_duration_seconds Flask HTTP request duration in seconds
# # TYPE memo_api_flask_http_request_duration_seconds histogram
# memo_api_flask_http_request_duration_seconds_bucket{...} ...
# # HELP memo_api_flask_http_request_total Total number of HTTP requests
# memo_api_flask_http_request_total{method="GET",status="200"} ...
```

---

## 5. Prometheus config

```yaml
# prometheus.yml
scrape_configs:
  - job_name: memo-api
    metrics_path: /metrics
    static_configs:
      - targets: ["memo-api:5000"]
    scrape_interval: 15s
```

---

## 6. 자주 보는 메트릭

| metric | 의미 |
| --- | --- |
| `flask_http_request_total{method, status, endpoint}` | 요청 수 |
| `flask_http_request_duration_seconds_*` | 응답 시간 (histogram) |
| `flask_http_request_exceptions_total{exception}` | 예외 발생 수 |
| `flask_exporter_info` | exporter 정보 |
| `process_cpu_seconds_total` | CPU 사용 |
| `process_resident_memory_bytes` | 메모리 사용 |

---

## 7. Custom metric

```python
metrics = PrometheusMetrics(app)

# Counter — 증가만
memos_created = metrics.counter(
    "memos_created_total",
    "메모 생성 횟수",
    labels={"env": lambda: app.config.get("ENV", "prod")},
)


@bp.post("/memos")
@memos_created
def create_memo():
    # ...
    return ok(...)


# Histogram — 분포
memo_size_bytes = metrics.histogram(
    "memo_size_bytes",
    "메모 본문 크기",
    labels={"env": lambda: app.config.get("ENV", "prod")},
)


# Gauge — 현재 값
active_users = metrics.gauge(
    "active_users",
    "현재 active user 수",
)
active_users.set_function(lambda: get_active_user_count())
```

---

## 8. Endpoint 별 자동 메트릭

```python
@bp.get("/memos")
@metrics.summary("memos_list_summary", "List 응답 시간")
def list_memos():
    return ...

@bp.post("/memos")
@metrics.counter("memos_create_count", "메모 생성")
@metrics.histogram("memos_create_time", "생성 시간")
def create_memo():
    return ...
```

---

## 9. metric 보호 — admin only

`/metrics` 가 외부 노출 = 모든 endpoint URL 노출 + APM 우회. **내부 망 only**:

```python
from werkzeug.middleware.proxy_fix import ProxyFix

@app.before_request
def restrict_metrics():
    if request.path == "/metrics":
        client_ip = request.headers.get("X-Forwarded-For", request.remote_addr)
        if not client_ip.startswith("10."):           # private IP only
            abort(403)
```

또는 Nginx 단에서 `/metrics` 외부 차단.

---

## 10. Grafana 대시보드 추천

- **Flask Dashboard for Prometheus** — flask-exporter 의 모든 메트릭 시각화
- **Python Application by JuliusV** — process / GC / memory

[Grafana Dashboards](https://grafana.com/grafana/dashboards/) 에서 검색.

---

## 11. Sentry — 에러 추적 (옵션)

```bash
pip install sentry-sdk[flask]
```

```python
# app/__init__.py
import sentry_sdk
from sentry_sdk.integrations.flask import FlaskIntegration

sentry_sdk.init(
    dsn=os.environ.get("SENTRY_DSN", ""),
    integrations=[FlaskIntegration()],
    traces_sample_rate=0.1,
    environment=app.config.get("ENV", "prod"),
    release="memo-api@1.0.0",
)
```

→ uncaught exception 자동 Sentry. 4xx 는 Sentry 안 가는 default.

---

## 12. 자주 막히는 함정

### 함정 1 — `/metrics` 외부 노출
endpoint URL / latency 노출 = APM 우회. **내부 망 only** (firewall / VPN / Nginx).

### 함정 2 — `/healthz` 가 무거움
DB 5초 query 등 = k8s probe fail → restart loop. **가벼운 ping**.

### 함정 3 — `readyz` 의 DB 끊김 시 무한 재시도
지나치게 strict 면 일시적 DB 끊김 = 모든 instance 빠짐. **graceful — 부분 fail OK**.

### 함정 4 — Prometheus scrape 너무 잦음
1초 = 부하. **15s ~ 60s**.

### 함정 5 — custom metric 의 label 폭증
`labels={"user_id": user.id}` = user 마다 별도 시계열 → Prometheus memory 폭발. **label 은 상수 / 작은 enum 만**.

### 함정 6 — Sentry 가 4xx alert
`BusinessException` 4xx 도 Sentry → 알람 폭주. **5xx 만 send** 또는 `before_send` filter.

### 함정 7 — gunicorn worker 마다 별도 metric (in-memory)
worker 별 시계열 분리. **`prometheus-flask-exporter` 의 `multiprocess` 모드** 또는 별도 process 의 sidecar exporter.

### 함정 8 — `app.before_request` hook 가 metric 매번 측정
이미 `PrometheusMetrics(app)` 가 자동 측정. 중복 X.

---

## 13. 다음

✅ Monitoring 끝.

→ [[09-logging-mdc]] — Logback / MDC / X-Request-Id 추적
