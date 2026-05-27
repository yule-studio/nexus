---
title: "Excel / CSV Import·Export — FastAPI"
kind: knowledge
status: current
created_at: 2026-05-27T00:10:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, api-design, recipe, excel, csv]
---

# Excel / CSV Import·Export — FastAPI

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/excel-csv-import-export|↗ Java 원본]]** · **[[../../python-flask/api-design/excel-csv-import-export|↗ Flask 변형]]**

> FastAPI `StreamingResponse` + `openpyxl` write-only + Celery (async job).

---

## 1-7-8. 무엇을 / 결과 / 보안

(Java 원본 [[../../spring/java-spring/api-design/excel-csv-import-export#1|§1, §7, §8]] 와 동일)

---

## 3. CSV Export — FastAPI StreamingResponse

```python
import csv
import io
from typing import Iterator
from fastapi import APIRouter, Depends
from fastapi.responses import StreamingResponse

from app.deps import SessionDep, require_role

router = APIRouter(prefix="/api/v1/exports")


@router.get("/orders.csv", dependencies=[Depends(require_role("ADMIN"))])
async def export_orders_csv(
    from_date: date, to_date: date, db: SessionDep,
) -> StreamingResponse:

    async def generate() -> AsyncIterator[bytes]:
        yield "﻿".encode("utf-8")              # UTF-8 BOM

        buf = io.StringIO()
        writer = csv.writer(buf)
        writer.writerow(["주문번호", "구매자", "금액", "상태", "주문일시"])
        yield buf.getvalue().encode("utf-8")
        buf.seek(0); buf.truncate()

        # async cursor stream from SQLAlchemy 2.x
        result = await db.stream(
            select(Order).where(Order.created_at.between(from_date, to_date))
        )
        async for row in result.scalars():
            writer.writerow([
                row.order_number, row.buyer_name, str(row.amount_krw),
                row.status, row.created_at.isoformat(),
            ])
            yield buf.getvalue().encode("utf-8")
            buf.seek(0); buf.truncate()

    return StreamingResponse(
        generate(),
        media_type="text/csv; charset=UTF-8",
        headers={"Content-Disposition": 'attachment; filename="orders.csv"'},
    )
```

> **FastAPI/async idiom**: `StreamingResponse(async_iterator)` — Starlette 가 chunk 단위 전송. async DB stream + chunk yield.

---

## 4. Excel Export — openpyxl write-only

```python
from openpyxl import Workbook


@router.get("/users.xlsx", dependencies=[Depends(require_role("ADMIN"))])
async def export_users_xlsx(db: SessionDep) -> StreamingResponse:
    wb = Workbook(write_only=True)
    ws = wb.create_sheet("Users")
    ws.append(["ID", "이메일", "이름", "가입일"])

    # async streaming
    result = await db.stream(select(User))
    async for user in result.scalars():
        ws.append([user.id, user.email, user.name, user.created_at.isoformat()])

    buf = io.BytesIO()
    wb.save(buf)
    buf.seek(0)
    return StreamingResponse(
        buf,
        media_type="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        headers={"Content-Disposition": 'attachment; filename="users.xlsx"'},
    )
```

---

## 5. Async Export Job — Celery or BackgroundTasks

큰 데이터 (100k+ 행) = response timeout 위험. Celery 가 표준.

```python
@router.post("/exports/orders")
async def request_export(
    payload: ExportRequest, user: CurrentUserDep, db: SessionDep,
) -> dict:
    job = ExportJob(
        requested_by=user.id, kind=payload.kind, params=payload.params.model_dump(),
        status="PENDING",
    )
    db.add(job)
    await db.commit()
    await db.refresh(job)

    # Celery 또는 BackgroundTasks
    export_orders_to_s3.delay(job.id, payload.kind, payload.params.model_dump())
    return {"jobId": job.id, "status": "PENDING"}


@shared_task(bind=True)
def export_orders_to_s3(self, job_id: int, kind: str, params: dict) -> None:
    # Celery worker — sync DB session (Flask 변형 §5 와 동일)
    # ...
    pass
```

---

## 6. Import — Celery worker

(Flask 변형 [[../../python-flask/api-design/excel-csv-import-export#6 CSV Import|§6]] 와 동일 — Celery worker 가 sync DB session 사용)

---

## 9. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/excel-csv-import-export#9 함정 모음|§9]] 의 10 함정 그대로)

### FastAPI 한정

### 함정 11 — `await db.stream(select(...))` 가 SQLAlchemy 2.x async 만
1.x sync 면 동작 X. **2.x async + asyncpg**.

### 함정 12 — `StreamingResponse(sync_iterator)` vs `async`
async endpoint 에서 sync generator OK 지만 generator 안 blocking IO 면 event loop block. **async generator + async DB**.

### 함정 13 — `openpyxl` write-only 가 sync
큰 파일 작성 시 event loop block. **`asyncio.to_thread` 또는 Celery worker** 권장.

---

## 10. 운영 체크리스트

(Java 원본 §10 + Python async)

---

## 11. 관련

- [[../../spring/java-spring/api-design/excel-csv-import-export|↗ Java 원본]]
- [[../../python-flask/api-design/excel-csv-import-export|↗ Flask 변형]]
- [[file-upload-s3]]
- [[api-design|↑ hub]]
