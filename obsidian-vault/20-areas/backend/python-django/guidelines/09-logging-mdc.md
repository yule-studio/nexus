---
title: "09. Logging — django-log-request-id + JSON log + Sentry"
kind: guide
status: current
created_at: 2026-05-26T12:45:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-django, guidelines, logging, request-id, mdc, json]
---

# 09. Logging — django-log-request-id + JSON log + Sentry

**[[guidelines|↑ Hub]]** ← [[08-monitoring]] · → 다음 [[10-devtools-hotreload]]

> 요청별 `request_id` 추적 + 구조화 (JSON) log + Sentry 통합. ELK / CloudWatch / Loki 친화.

---

## 1. 기본 logging — `LOGGING` dict

```python
# config/settings/base.py
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "verbose": {
            "format": "{asctime} [{levelname}] {name}: {message}",
            "style": "{",
        },
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "verbose",
        },
    },
    "root": {"handlers": ["console"], "level": "INFO"},
    "loggers": {
        "django": {"handlers": ["console"], "level": "INFO", "propagate": False},
        "django.db.backends": {"level": "WARNING"},        # SQL log off
        "apps": {"level": "DEBUG" if DEBUG else "INFO"},
    },
}
```

```python
# 사용
import logging
log = logging.getLogger(__name__)


def view(request):
    log.info("create memo: title=%s", payload["title"])
    try:
        memo = service.create(...)
    except Exception:
        log.exception("memo create failed")
        raise
    log.info("memo created: id=%d", memo.id)
```

---

## 2. django-log-request-id — request_id 자동 추적

```bash
pip install django-log-request-id
```

```python
# config/settings/base.py
MIDDLEWARE = [
    "log_request_id.middleware.RequestIDMiddleware",       # 가장 위
    ...
]

LOG_REQUEST_ID_HEADER = "HTTP_X_REQUEST_ID"               # 외부에서 받은 ID 우선
GENERATE_REQUEST_ID_IF_NOT_IN_HEADER = True
REQUEST_ID_RESPONSE_HEADER = "X-Request-Id"               # 응답에도 노출
LOG_REQUESTS = True                                        # access log

LOGGING["filters"] = {
    "request_id": {"()": "log_request_id.filters.RequestIDFilter"},
}
LOGGING["formatters"]["verbose"]["format"] = (
    "{asctime} [{levelname}] request_id={request_id} {name}: {message}"
)
LOGGING["handlers"]["console"]["filters"] = ["request_id"]
```

→ 모든 log 라인에 `request_id=req-xxxx` 자동 첨부.

---

## 3. JSON log — python-json-logger

```bash
pip install python-json-logger
```

```python
LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "filters": {
        "request_id": {"()": "log_request_id.filters.RequestIDFilter"},
    },
    "formatters": {
        "json": {
            "()": "pythonjsonlogger.jsonlogger.JsonFormatter",
            "format": (
                "%(asctime)s %(levelname)s %(name)s "
                "%(request_id)s %(message)s "
                "%(module)s %(funcName)s %(lineno)d"
            ),
            "rename_fields": {"levelname": "level", "asctime": "timestamp"},
        },
        "verbose": {
            "format": "{asctime} [{levelname}] {name}: {message}",
            "style": "{",
        },
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "json" if not DEBUG else "verbose",
            "filters": ["request_id"],
        },
    },
    "root": {"handlers": ["console"], "level": "INFO"},
    "loggers": {
        "django": {"handlers": ["console"], "level": "INFO", "propagate": False},
        "django.db.backends": {"level": "WARNING"},
        "apps": {"level": "DEBUG" if DEBUG else "INFO"},
    },
}
```

→ dev = 사람 친화 / prod = JSON.

prod JSON 출력:
```json
{"timestamp":"2026-05-26 12:45:00","level":"INFO","name":"apps.memos.views",
 "request_id":"req-abc1234","message":"memo created","module":"views","funcName":"perform_create","lineno":42}
```

→ ELK / CloudWatch 가 자동 색인.

---

## 4. user_id 같은 context 자동 첨부

```python
# apps/common/middleware.py
import threading

_local = threading.local()


class UserContextMiddleware:
    """현재 요청의 user 를 thread-local 에 저장."""

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        _local.user_id = getattr(request.user, "id", None) if hasattr(request, "user") else None
        try:
            return self.get_response(request)
        finally:
            _local.user_id = None


class UserContextFilter:
    def filter(self, record):
        record.user_id = getattr(_local, "user_id", None)
        return True
```

```python
# settings/base.py
MIDDLEWARE += ["apps.common.middleware.UserContextMiddleware"]

LOGGING["filters"]["user_context"] = {"()": "apps.common.middleware.UserContextFilter"}
LOGGING["handlers"]["console"]["filters"] = ["request_id", "user_context"]

# JSON formatter 의 format 에 %(user_id)s 추가
```

→ 모든 log 에 `user_id` 자동.

---

## 5. SQL log — `django.db.backends`

```python
LOGGING["loggers"]["django.db.backends"] = {
    "handlers": ["console"],
    "level": "DEBUG" if env.bool("SQL_LOG", default=False) else "WARNING",
    "propagate": False,
}
```

```bash
SQL_LOG=true python manage.py runserver
```

→ 모든 SQL log. dev 디버깅 용. 운영 WARN.

---

## 6. access log — django-log-request-id 의 `LOG_REQUESTS`

`LOG_REQUESTS=True` → 모든 요청의 method / path / status / duration 자동.

또는 custom middleware:

```python
import time
import logging

log = logging.getLogger("access")


class AccessLogMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        start = time.monotonic()
        response = self.get_response(request)
        dur_ms = int((time.monotonic() - start) * 1000)
        log.info(
            "access",
            extra={
                "method": request.method,
                "path": request.path,
                "status": response.status_code,
                "dur_ms": dur_ms,
            },
        )
        return response
```

→ JSON formatter 와 결합 시 `{"event":"access","method":"GET",...}`.

---

## 7. Sentry 의 log 통합

[[08-monitoring#7 Sentry|08 §7]] 의 `DjangoIntegration` 이 이미 logger 통합:

```python
import sentry_sdk
from sentry_sdk.integrations.logging import LoggingIntegration


sentry_sdk.init(
    dsn=env("SENTRY_DSN"),
    integrations=[
        DjangoIntegration(),
        LoggingIntegration(
            level=logging.INFO,                            # breadcrumb level
            event_level=logging.ERROR,                     # Sentry event level
        ),
    ],
)
```

→ ERROR 이상이 Sentry event. INFO 는 breadcrumb.

---

## 8. log rotation — 운영

stdout/stderr → docker/k8s 가 수집 (권장). 파일 로깅이 필요하면 `RotatingFileHandler`:

```python
LOGGING["handlers"]["file"] = {
    "class": "logging.handlers.RotatingFileHandler",
    "filename": "/var/log/django/app.log",
    "maxBytes": 10 * 1024 * 1024,                          # 10MB
    "backupCount": 10,
    "formatter": "json",
}
```

> **stdout 우선** (k8s / docker / Loki 친화). 파일 로깅은 옛 운영.

---

## 9. structlog (선택) — 더 깔끔

```bash
pip install structlog
```

structlog 도 동일 가능. Django 의 `logging` 과 통합:

```python
import structlog


structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ],
    wrapper_class=structlog.stdlib.BoundLogger,
    logger_factory=structlog.stdlib.LoggerFactory(),
)


# 사용
log = structlog.get_logger()
log.info("memo created", memo_id=memo.id, title=memo.title)
```

→ FastAPI 의 [[../../python-fastapi/guidelines/09-logging-mdc|FastAPI 09]] 와 비슷한 사용감.

---

## 10. 자주 막히는 함정

### 함정 1 — `django-log-request-id` middleware 위치
**가장 위** (Security 보다도 위 권장). 아니면 일부 log 가 request_id 없음.

### 함정 2 — `propagate=True` (default)
같은 log 가 root 로 전파 → 중복 출력. 명시적 propagate=False.

### 함정 3 — `django.db.backends` 운영 INFO
모든 SQL log → 폭증. WARN.

### 함정 4 — JSON formatter 의 `rename_fields` 누락
`asctime` → `timestamp` 같은 표준 필드명으로. ELK index template 와 일치.

### 함정 5 — thread-local user_context 누락
gunicorn worker 의 다음 요청에 이전 user_id 가 leak 가능. `finally _local.user_id=None` 필수.

### 함정 6 — log 의 sensitive data
password / token / PII 가 log → 노출. filter / redact.

### 함정 7 — Sentry 의 ERROR + 4xx 알람 폭주
`AuthenticationFailed` 같은 4xx 가 ERROR 로 → 알람. `before_send` filter.

### 함정 8 — Cloud 환경의 multi-line stack trace
docker / CloudWatch 가 multi-line 분리. **JSON formatter** 가 single line.

### 함정 9 — `extra={}` 의 reserved key 충돌
`record.message` 같은 reserved 와 충돌 시 silent drop. 다른 이름.

### 함정 10 — `LOG_REQUESTS=True` 의 access log 중복
custom AccessLogMiddleware 와 동시 → 중복. 하나만.

---

## 11. 다음

→ [[10-devtools-hotreload]] — runserver / pytest-django / debug-toolbar
