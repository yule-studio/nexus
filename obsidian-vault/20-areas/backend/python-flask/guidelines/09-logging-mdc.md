---
title: "09. Logging — Python logging + Request ID + JSON log"
kind: guide
status: current
created_at: 2026-05-27T13:55:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-flask, guidelines, logging, request-id, json-log]
---

# 09. Logging — Python logging + Request ID + JSON log

**[[guidelines|↑ Hub]]** ← [[08-monitoring]] · → 다음 [[10-devtools-hotreload]]

> Python 표준 `logging` 모듈 + Flask 의 `flask.g` 로 request ID 추적 + 운영은 JSON log.

---

## 1. Python logging 기본

```python
import logging

log = logging.getLogger(__name__)


@bp.post("/memos")
def create_memo():
    log.info("create memo: title=%s", request.json.get("title"))
    log.debug("payload: %s", request.json)

    try:
        memo = memo_service.create(...)
        log.info("memo created: id=%d", memo.id)
    except Exception as e:
        log.error("memo create failed", exc_info=True)
        raise

    return ok(memo)
```

### 1.1 logger 명명 — `__name__`

```python
log = logging.getLogger(__name__)
```

→ module path (`app.api.memos`) 가 logger name. 계층 구조 자동.

### 1.2 placeholder vs f-string

```python
# ✅ placeholder — DEBUG off 면 evaluation 안 함
log.debug("user = %s", user)

# ❌ f-string — DEBUG off 여도 user.toString() 호출
log.debug(f"user = {user}")
```

### 1.3 `exc_info` / `stack_info`

```python
try:
    ...
except Exception as e:
    log.error("failed: %s", str(e), exc_info=True)        # stack trace 포함
    # 또는
    log.exception("failed")                                # log.error(..., exc_info=True) 단축
```

→ `log.exception` 은 ERROR level 자동. 다른 level 이면 `exc_info=True`.

---

## 2. Flask 의 logging 설정

```python
# app/__init__.py
import logging
import sys
from logging.config import dictConfig


def configure_logging(app: Flask) -> None:
    dictConfig({
        "version": 1,
        "formatters": {
            "default": {
                "format": "%(asctime)s [%(levelname)s] [%(request_id)s] %(name)s: %(message)s",
            },
        },
        "filters": {
            "request_id": {
                "()": "app.common.logging_filter.RequestIdFilter",
            },
        },
        "handlers": {
            "console": {
                "class": "logging.StreamHandler",
                "formatter": "default",
                "filters": ["request_id"],
                "stream": "ext://sys.stdout",
            },
        },
        "loggers": {
            "app": {                                              # 우리 패키지
                "level": "DEBUG" if app.debug else "INFO",
                "handlers": ["console"],
                "propagate": False,
            },
            "sqlalchemy.engine": {                                # SQL log
                "level": "INFO" if app.debug else "WARNING",
                "handlers": ["console"],
                "propagate": False,
            },
        },
        "root": {
            "level": "INFO",
            "handlers": ["console"],
        },
    })


def create_app(...) -> Flask:
    app = Flask(__name__)
    # ...
    configure_logging(app)
    return app
```

---

## 3. Request ID 추적 — `flask.g` + Filter

### 3.1 RequestIdMiddleware

```python
# app/common/logging_filter.py
import logging
import uuid
from flask import g


class RequestIdFilter(logging.Filter):
    """log record 에 request_id 추가."""

    def filter(self, record: logging.LogRecord) -> bool:
        record.request_id = getattr(g, "request_id", "-") if g else "-"
        return True
```

### 3.2 before_request 에서 ID 생성

```python
# app/__init__.py
import uuid
from flask import g, request


def create_app(...) -> Flask:
    app = Flask(__name__)
    # ...

    @app.before_request
    def assign_request_id():
        incoming = request.headers.get("X-Request-Id")
        rid = incoming if incoming else f"req-{uuid.uuid4().hex[:8]}"
        g.request_id = rid

    @app.after_request
    def attach_request_id(response):
        if hasattr(g, "request_id"):
            response.headers["X-Request-Id"] = g.request_id
        return response

    return app
```

→ 응답 헤더에도 `X-Request-Id` 노출. 클라가 issue report 시 함께.

### 3.3 결과

```
2026-05-27 13:55:00 [INFO] [req-abc1234] app.api.memos: create memo: title=hello
2026-05-27 13:55:00 [INFO] [req-abc1234] app.services.memo_service: memo created: id=1
2026-05-27 13:55:01 [INFO] [req-abc1234] app.api.memos: list memos: count=5
```

→ `grep "req-abc1234"` 로 한 요청의 전체 흐름.

---

## 4. user_id 추가

```python
# JWT verify 후
@app.before_request
def attach_user_id():
    if has_jwt_token():
        try:
            verify_jwt_in_request()
            g.user_id = get_jwt_identity()
        except Exception:
            pass
```

filter 도 확장:

```python
class RequestIdFilter(logging.Filter):
    def filter(self, record):
        record.request_id = getattr(g, "request_id", "-") if g else "-"
        record.user_id = getattr(g, "user_id", "-") if g else "-"
        return True
```

formatter 도:

```python
"format": "%(asctime)s [%(levelname)s] [req=%(request_id)s user=%(user_id)s] %(name)s: %(message)s",
```

---

## 5. JSON log — 운영

운영은 사람이 보는 text 보다 **JSON** 이 ELK / Loki / CloudWatch 검색 효율적.

```bash
pip install python-json-logger
```

```python
# app/__init__.py
from pythonjsonlogger import jsonlogger

def configure_logging(app: Flask) -> None:
    formatter = jsonlogger.JsonFormatter(
        "%(asctime)s %(levelname)s %(name)s %(message)s %(request_id)s %(user_id)s",
        rename_fields={"asctime": "timestamp", "levelname": "level", "name": "logger"},
    )
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(formatter)
    handler.addFilter(RequestIdFilter())

    for logger_name in ["app", "sqlalchemy.engine"]:
        logger = logging.getLogger(logger_name)
        logger.addHandler(handler)
        logger.setLevel(logging.INFO)
        logger.propagate = False
```

### 5.1 출력 예

```json
{
  "timestamp": "2026-05-27 13:55:00,123",
  "level": "INFO",
  "logger": "app.api.memos",
  "message": "create memo: title=hello",
  "request_id": "req-abc1234",
  "user_id": "1"
}
```

→ CloudWatch Logs Insights 의 `fields @timestamp, request_id, message | filter request_id = "req-abc1234"`.

---

## 6. Access log — 모든 HTTP 요청

```python
# app/__init__.py
@app.before_request
def access_log_start():
    g.access_log_start = time.monotonic()


@app.after_request
def access_log_end(response):
    dur_ms = int((time.monotonic() - g.access_log_start) * 1000)
    access_log = logging.getLogger("access")
    access_log.info(
        "%s %s status=%d dur_ms=%d",
        request.method, request.path, response.status_code, dur_ms,
    )
    return response
```

---

## 7. SQL log — SQLAlchemy

```python
# application.yml
SQLALCHEMY_ECHO = True                                 # dev/staging 만
```

또는 logger 단:

```python
logging.getLogger("sqlalchemy.engine").setLevel(logging.INFO)        # SQL 자체
logging.getLogger("sqlalchemy.engine").setLevel(logging.DEBUG)       # SQL + bind param
```

→ 운영은 `WARN` (성능 / 노이즈).

---

## 8. Sentry 통합 — exception → Sentry + MDC

[[08-monitoring#11 Sentry|08 §11]] 참고. Sentry SDK 가 `extra` / `tags` 로 MDC 정보 자동 첨부:

```python
import sentry_sdk

@app.before_request
def sentry_context():
    if hasattr(g, "request_id"):
        sentry_sdk.set_tag("request_id", g.request_id)
    if hasattr(g, "user_id"):
        sentry_sdk.set_user({"id": g.user_id})
```

→ Sentry 이벤트에 자동 첨부.

---

## 9. 자주 막히는 함정

### 함정 1 — `g` 가 request context 끝나면 사라짐
async task / Celery worker 에선 새 context 필요. **`with app.app_context():`**.

### 함정 2 — `f-string` SQL log 폭증
`log.debug(f"user = {user}")` = DEBUG off 여도 `user.__repr__()` 호출. **`log.debug("user = %s", user)`** placeholder.

### 함정 3 — logger 가 root 로 propagate
`logger.propagate = True` (default) → root logger 도 처리 → 중복 출력. **`False`**.

### 함정 4 — gunicorn worker 의 access log 와 Flask access log 중복
gunicorn `--access-logfile -` 와 우리 `@app.after_request` 둘 다. 한쪽만 활성.

### 함정 5 — `request_id` filter 가 g context 밖에서 호출
모든 log record 가 filter 통과 — 외부 logger (sqlalchemy 등) 의 record 는 g context 없을 수 있음. **`getattr(g, "x", "-")` 안전 처리**.

### 함정 6 — JSON log + Docker 의 multi-line stack trace
default stack trace = multi-line → docker 가 분리된 로그로 인식. **`stack_info_format="single_line"`** 또는 `traceback.format_exc()` 명시 변환.

### 함정 7 — log file 에 sensitive data
password / token / 카드번호 = 로그 누수. **filter / redact** 또는 logging level 검토.

### 함정 8 — Python 한정: thread-local g 가 gunicorn worker 마다 격리
한 worker 의 g 가 다른 worker 와 무관. (정상 — request 단위)

### 함정 9 — `dictConfig` 의 `disable_existing_loggers`
default True 면 import-time 에 만든 모든 logger 가 비활성. **`disable_existing_loggers: False`** 권장.

### 함정 10 — `app.logger` vs `logging.getLogger(__name__)`
Flask 의 `app.logger` 는 별도 logger. 본 가이드는 **모듈 단 `getLogger(__name__)`** 권장 (일관성).

---

## 10. 다음

✅ Logging + Request ID 끝.

→ [[10-devtools-hotreload]] — debug mode / `flask --reload` / Werkzeug debugger
