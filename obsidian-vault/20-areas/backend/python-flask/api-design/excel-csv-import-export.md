---
title: "Excel / CSV Import·Export — Flask"
kind: knowledge
status: current
created_at: 2026-05-26T17:40:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-flask, api-design, recipe, excel, csv]
---

# Excel / CSV Import·Export — Flask

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/excel-csv-import-export|↗ Java 원본]]**

> `openpyxl` (Excel write-only mode), `csv` stdlib 기반.

---

## 1. 무엇을 / 1.1 용량별 / 7. 결과 / 8. 보안

(Java 원본 [[../../spring/java-spring/api-design/excel-csv-import-export#1|§1, §7, §8]] 와 동일)

---

## 2. 의존성

```python
pip install openpyxl boto3 celery[redis]
```

---

## 3. CSV Export — Flask streaming

```python
import csv
from io import StringIO
from flask import Response, stream_with_context


@bp.get("/exports/orders.csv")
@role_required("ADMIN")
def export_orders_csv():
    from_date = request.args.get("from")
    to_date = request.args.get("to")

    def generate():
        # UTF-8 BOM (엑셀 한글 깨짐 방지)
        yield "﻿"

        buf = StringIO()
        writer = csv.writer(buf)
        writer.writerow(["주문번호", "구매자", "금액", "상태", "주문일시"])
        yield buf.getvalue()
        buf.seek(0); buf.truncate()

        # Cursor stream from SQLAlchemy
        for row in order_repo.stream_between(from_date, to_date):
            writer.writerow([
                row.order_number, row.buyer_name, str(row.amount_krw),
                row.status, row.created_at.isoformat(),
            ])
            yield buf.getvalue()
            buf.seek(0); buf.truncate()

    response = Response(stream_with_context(generate()), mimetype="text/csv; charset=UTF-8")
    response.headers["Content-Disposition"] = 'attachment; filename="orders.csv"'
    return response
```

> **Flask idiom**: `stream_with_context(generator)` — request context 가 살아있는 채로 generator 평가.

---

## 4. Excel Export — openpyxl write-only

```python
from openpyxl import Workbook


@bp.get("/exports/users.xlsx")
@role_required("ADMIN")
def export_users_xlsx():
    wb = Workbook(write_only=True)                        # 메모리 streaming
    ws = wb.create_sheet("Users")
    ws.append(["ID", "이메일", "이름", "가입일"])

    for u in user_repo.stream_all():
        ws.append([u.id, u.email, u.name, u.created_at.isoformat()])

    buf = io.BytesIO()
    wb.save(buf)
    buf.seek(0)
    return Response(
        buf.read(),
        mimetype="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        headers={"Content-Disposition": 'attachment; filename="users.xlsx"'},
    )
```

> **Python 한정**: `Workbook(write_only=True)` 가 SXSSF 동등 — 100만 행도 메모리 안 폭발.

---

## 5. Excel Export — Async (Celery + S3)

```python
@shared_task(bind=True)
def export_orders_to_s3(self, job_id: str, kind: str, params: dict) -> None:
    job = export_repo.find_by_id(job_id)
    job.status = "RUNNING"
    job.started_at = datetime.now(timezone.utc)
    db.session.commit()

    try:
        s3_key = f"exports/{job_id}.xlsx"
        row_count = _write_excel_to_s3(kind, params, s3_key)
        job.status = "SUCCEEDED"
        job.s3_key = s3_key
        job.row_count = row_count
        job.completed_at = datetime.now(timezone.utc)
    except Exception as e:
        job.status = "FAILED"
        job.error = str(e)[:500]
    db.session.commit()
```

---

## 6. CSV Import — 행 단위 검증

```python
@shared_task(bind=True)
def import_products_csv(self, job_id: str) -> None:
    job = import_repo.find_by_id(job_id)
    try:
        with s3_client.get_object(Bucket=job.bucket, Key=job.s3_key)["Body"] as f:
            reader = csv.reader(io.TextIOWrapper(f, encoding="utf-8"))
            next(reader)                                  # header skip

            processed = succeeded = failed = 0
            for row_num, row in enumerate(reader, start=1):
                processed += 1
                try:
                    product_import_service.import_row(job.requested_by, row)
                    succeeded += 1
                except Exception as e:
                    failed += 1
                    import_error_repo.save(ImportError(
                        job_id=job_id, row_number=row_num,
                        message=str(e)[:500], raw_row=",".join(row),
                    ))

                if processed % 100 == 0:
                    job.processed = processed
                    job.succeeded = succeeded
                    job.failed = failed
                    db.session.commit()

            job.status = "SUCCEEDED"
            job.processed = processed
            job.succeeded = succeeded
            job.failed = failed
    except Exception as e:
        job.status = "FAILED"
        job.error = str(e)[:500]
    db.session.commit()
```

---

## 9. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/excel-csv-import-export#9 함정 모음|§9]] 의 10 함정 그대로)

### Python 한정

### 함정 11 — `openpyxl` write-only 의 `ws.append` 만 가능
`ws.cell(row, col)` 같은 random access X. **순차 write 만**.

### 함정 12 — `csv` stdlib 의 UTF-8 BOM
`﻿` 직접 prepend 또는 `csv.writer(..., encoding="utf-8-sig")` (Python 3.12+ writer 옵션 없음 → file open 시 `encoding="utf-8-sig"`).

### 함정 13 — CSV injection
악의 cell `=cmd|'/c calc'!A1` 가 Excel 에서 실행. 화이트리스트 / quote prefix:
```python
def safe_cell(v: str) -> str:
    return f"'{v}" if v and v[0] in ("=", "+", "-", "@") else v
```

---

## 10. 운영 체크리스트

(Java 원본 §10 그대로 + Python: write_only / BOM / CSV injection)

---

## 11. 관련

- [[../../spring/java-spring/api-design/excel-csv-import-export|↗ Java 원본]]
- [[file-upload-s3]] — Import 의 업로드
- [[api-design|↑ hub]]
