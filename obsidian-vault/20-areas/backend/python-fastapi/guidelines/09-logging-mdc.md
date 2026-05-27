---
title: "09. Logging — structlog + asgi-correlation-id"
kind: guide
status: current
created_at: 2026-05-27T16:25:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, guidelines, logging, structlog, correlation-id]
---

# 09. Logging — structlog + asgi-correlation-id

**[[guidelines|↑ Hub]]** ← [[08-monitoring]] · → 다음 [[10-devtools-hotreload]]

> **structlog** = structured logging (JSON 친화) + **asgi-correlation-id** = X-Request-Id 자동 추적.

---

## 1. 표준 logging — 시작점

```python
import logging

log = logging.getLogger(__name__)


@router.post("/memos")
async def create_memo(payload: CreateMemoRequest):
    log.info("create memo: title=%s", payload.title)
    try:
        memo = await memo_service.create(...)
        log.info("memo created: id=%d", memo.id)
    except Exception:
        log.exception("memo create failed")
        raise
    return memo
```

기본 logging 도 OK 지만 운영 = **structlog (JSON)** 권장.

---

## 2. structlog 설치 + 설정

```bash
pip install structlog asgi-correlation-id
```

```python
# app/logging_config.py
import logging
import structlog
import sys


def configure_logging(debug: bool = False) -> None:
    log_level = logging.DEBUG if debug else logging.INFO

    # stdlib logging 기본
    logging.basicConfig(
        format="%(message)s",
        stream=sys.stdout,
        level=log_level,
    )

    # structlog
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,                        # request context 자동 첨부
            structlog.stdlib.filter_by_level,
            structlog.stdlib.add_logger_name,
            structlog.stdlib.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            structlog.processors.UnicodeDecoder(),

            # 환경별 — dev = 사람 친화, prod = JSON
            structlog.dev.ConsoleRenderer() if debug
            else structlog.processors.JSONRenderer(),
        ],
        wrapper_class=structlog.stdlib.BoundLogger,
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
        cache_logger_on_first_use=True,
    )
```

```python
# app/main.py
from .logging_config import configure_logging
from .config import settings

configure_logging(debug=settings.DEBUG)
```

---

## 3. structlog 사용

```python
import structlog

log = structlog.get_logger(__name__)


@router.post("/memos")
async def create_memo(payload: CreateMemoRequest):
    log.info("create memo", title=payload.title)        # kwargs = structured fields
    try:
        memo = await memo_service.create(...)
        log.info("memo created", memo_id=memo.id)
    except Exception as e:
        log.error("memo create failed", error=str(e), exc_info=True)
        raise
    return memo
```

### 3.1 출력 — dev 모드 (ConsoleRenderer)

```
2026-05-27T16:25:00 [info     ] create memo                    [app.api.memos] title=hello
2026-05-27T16:25:00 [info     ] memo created                   [app.api.memos] memo_id=1
```

### 3.2 출력 — prod 모드 (JSONRenderer)

```json
{"timestamp": "2026-05-27T16:25:00.123Z", "level": "info", "logger": "app.api.memos",
 "event": "create memo", "title": "hello", "request_id": "req-abc1234"}
```

→ ELK / CloudWatch / Loki 에서 `event:"create memo"` 또는 `request_id:"req-abc1234"` 검색.

---

## 4. asgi-correlation-id — request_id 자동 추적

```python
# app/main.py
from asgi_correlation_id import CorrelationIdMiddleware


app.add_middleware(
    CorrelationIdMiddleware,
    header_name="X-Request-Id",
    update_request_header=True,                  # request header 에 ID 주입
    generator=lambda: f"req-{uuid.uuid4().hex[:8]}",
)
```

### 4.1 structlog 통합

asgi-correlation-id 가 `contextvars` 에 ID 저장 → structlog 의 `merge_contextvars` 가 자동으로 모든 log 에 첨부.

```python
# logging_config.py 의 processors 에 이미 있음
structlog.contextvars.merge_contextvars,
```

→ 모든 log 에 `request_id` 자동.

### 4.2 결과

```json
{"event": "create memo", "title": "hello", "request_id": "req-abc1234", ...}
{"event": "memo created", "memo_id": 1, "request_id": "req-abc1234", ...}
```

→ `request_id` 로 한 요청의 전체 흐름 grep.

---

## 5. user_id 도 자동 추가

```python
# app/deps.py 의 get_current_user 안
from asgi_correlation_id.context import correlation_id
import structlog


async def get_current_user(token: TokenDep, db: SessionDep) -> User:
    user = ...                                          # JWT 검증 + DB lookup

    # contextvars 에 user_id 추가
    structlog.contextvars.bind_contextvars(user_id=str(user.id))
    return user
```

→ 이후 log 에 `user_id` 자동.

---

## 6. SQLAlchemy SQL log

```python
# config.py
class Settings(BaseSettings):
    SQL_LOG: bool = False                        # dev 만 True


# db.py
engine = create_async_engine(
    settings.DATABASE_URL,
    echo=settings.SQL_LOG,                       # True 면 모든 SQL log
)


# 또는 logger 단
logging.getLogger("sqlalchemy.engine").setLevel(
    logging.INFO if settings.SQL_LOG else logging.WARNING
)
```

---

## 7. access log — middleware

`uvicorn` 자체 access log:

```bash
uvicorn app.main:app --log-config logging.yaml --access-log
```

또는 custom middleware (구조화):

```python
import time
from asgi_correlation_id.context import correlation_id

import structlog
log = structlog.get_logger("access")


@app.middleware("http")
async def access_log_middleware(request: Request, call_next):
    start = time.monotonic()
    try:
        response = await call_next(request)
        return response
    finally:
        dur_ms = int((time.monotonic() - start) * 1000)
        log.info(
            "access",
            method=request.method,
            path=request.url.path,
            status=response.status_code if 'response' in locals() else 500,
            dur_ms=dur_ms,
            client_ip=request.client.host if request.client else None,
        )
```

---

## 8. Sentry 통합 — exception + context

[[08-monitoring#7 Sentry|08 §7]] 참고. structlog 와 통합:

```python
import sentry_sdk
from sentry_sdk.integrations.logging import LoggingIntegration

sentry_sdk.init(
    dsn=settings.SENTRY_DSN,
    integrations=[
        LoggingIntegration(
            level=logging.INFO,                  # breadcrumb level
            event_level=logging.ERROR,           # Sentry event level
        ),
    ],
)
```

→ ERROR 이상 자동 Sentry. INFO 는 breadcrumb (event 발생 시 함께 첨부).

---

## 9. 자주 막히는 함정

### 함정 1 — `logging` 와 `structlog` 혼용
일관성 X. **structlog 만 사용** 또는 stdlib logging 만.

### 함정 2 — `bind_contextvars` 가 request scope 끝나면 stale
async context 라 자동 정리되지만 BackgroundTasks 안에선 새 context 명시.

### 함정 3 — JSON log 의 multi-line stack trace
docker / CloudWatch 가 multi-line 분리. **`format_exc_info` processor** 가 single field 로.

### 함정 4 — `processors=[ConsoleRenderer, JSONRenderer]` 둘 다
순서대로 적용 → ConsoleRenderer 가 먼저 처리하면 JSON 못 함. **하나만**.

### 함정 5 — `cache_logger_on_first_use=True` + log level 변경 안 됨
runtime 에 level 변경 시 cache 무효화 필요. dev 환경 외엔 OK.

### 함정 6 — asgi-correlation-id 가 middleware 순서 영향
가장 outer 에 등록 권장 (모든 middleware / endpoint 에 ID 전파).

### 함정 7 — SQL log 운영
INSERT/UPDATE 마다 log = 폭증. **운영 `WARN`**.

### 함정 8 — `log.info(...)` 의 placeholder vs f-string
structlog 는 message 첫 인자 + kwargs. **`log.info("event", field=value)`** 형식 권장.

### 함정 9 — `correlation_id.get()` 가 None
middleware 전에 호출 시 None. middleware 등록 확인.

### 함정 10 — sensitive data in log
password / token / PII = 노출. filter / redact.

---

## 10. 다음

✅ Logging + Request ID 끝.

→ [[10-devtools-hotreload]] — uvicorn --reload / fastapi dev / pytest-asyncio
