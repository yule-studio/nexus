---
title: "파일/이미지 업로드 (S3 presigned URL) — FastAPI"
kind: knowledge
status: current
created_at: 2026-05-27T00:20:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, api-design, recipe, file-upload, s3]
---

# 파일/이미지 업로드 (S3 presigned URL) — FastAPI

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/file-upload-s3|↗ Java 원본]]** · **[[../../python-flask/api-design/file-upload-s3|↗ Flask 변형]]**

> `aioboto3` (async) 또는 sync `boto3` + `asyncio.to_thread`.

---

## 1-2-7. 왜 presigned / API spec / CORS

(Java 원본 [[../../spring/java-spring/api-design/file-upload-s3#1|§1, §2, §7]] 와 동일)

---

## 3-5. 도메인 / DB / UseCase

(Flask 변형 [[../../python-flask/api-design/file-upload-s3#3 도메인|§3-§5]] 의 SQLAlchemy 모델 동일 — async 만 다름)

---

## 4. S3 Service — aioboto3

```bash
pip install aioboto3
```

```python
# app/services/s3_file_service.py
import aioboto3
from typing import Optional
from datetime import datetime, timedelta, timezone

from app.config import settings


class S3FileService:
    def __init__(self) -> None:
        self.session = aioboto3.Session()
        self.bucket = settings.S3_BUCKET
        self.upload_ttl = 5 * 60
        self.download_ttl = 15 * 60
        self.cdn_base_url = settings.S3_CDN_BASE_URL

    async def build_upload_url(self, key: str, content_type: str, size_bytes: int) -> dict:
        async with self.session.client("s3", region_name=settings.S3_REGION) as s3:
            url = await s3.generate_presigned_url(
                "put_object",
                Params={
                    "Bucket": self.bucket,
                    "Key": key,
                    "ContentType": content_type,
                    "ContentLength": size_bytes,
                },
                ExpiresIn=self.upload_ttl,
            )
        return {
            "uploadUrl": url,
            "bucket": self.bucket,
            "key": key,
            "expiresAt": (datetime.now(timezone.utc) + timedelta(seconds=self.upload_ttl)).isoformat(),
        }

    async def build_download_url(self, key: str) -> str:
        if self.cdn_base_url:
            return f"{self.cdn_base_url}/{key}"
        async with self.session.client("s3") as s3:
            return await s3.generate_presigned_url(
                "get_object",
                Params={"Bucket": self.bucket, "Key": key},
                ExpiresIn=self.download_ttl,
            )

    async def verify(self, key: str) -> dict:
        async with self.session.client("s3") as s3:
            try:
                return await s3.head_object(Bucket=self.bucket, Key=key)
            except s3.exceptions.NoSuchKey:
                raise NotFoundException("S3 에 파일이 없습니다.")
```

> **`aioboto3` idiom**: `async with self.session.client(...)` — 매 호출마다 client. 또는 lifespan 단 single instance.

---

## 6. Controller + 후처리 (BackgroundTasks)

```python
from fastapi import APIRouter, BackgroundTasks
from PIL import Image
import io


@router.post("/files/presigned-url", response_model=CommonResponse[PresignedResponse])
async def presigned(
    payload: PresignedRequest, user: CurrentUserDep, db: SessionDep,
) -> CommonResponse[PresignedResponse]:
    result = await issue_upload_url(
        db, user.id, FileKind[payload.kind],
        payload.contentType, payload.sizeBytes,
        payload.originalFilename,
    )
    return CommonResponse.success(ResponseCode.OK, "URL 발급", PresignedResponse(**result))


@router.post("/files/{file_id}/complete")
async def complete(
    file_id: int, user: CurrentUserDep, db: SessionDep, bg: BackgroundTasks,
):
    meta = await complete_upload(db, file_id, user.id)

    # 후처리 — 작은 작업이면 BackgroundTasks, 큰 작업이면 Celery
    if meta.kind == FileKind.PRODUCT_IMAGE.value:
        bg.add_task(_generate_thumbnails, meta.s3_bucket, meta.s3_key)

    return CommonResponse.success(ResponseCode.OK, "업로드 완료")


async def _generate_thumbnails(bucket: str, key: str) -> None:
    """async wrapper — PIL 은 sync 라 to_thread."""
    import asyncio

    async with s3_session.client("s3") as s3:
        obj = await s3.get_object(Bucket=bucket, Key=key)
        img_bytes = await obj["Body"].read()

    # CPU bound — thread pool
    def _resize_and_upload():
        for size in (256, 512, 1024):
            img = Image.open(io.BytesIO(img_bytes))
            img.thumbnail((size, size))
            buf = io.BytesIO()
            img.save(buf, format="WEBP", quality=85)
            buf.seek(0)
            # boto3 sync — to_thread 안에서
            boto3_client.put_object(
                Bucket=bucket,
                Key=key.replace(key.split("/")[0], f"thumbnails/{size}", 1),
                Body=buf.getvalue(),
                ContentType="image/webp",
            )

    await asyncio.to_thread(_resize_and_upload)
```

> **`asyncio.to_thread(sync_fn)`** — CPU heavy (PIL resize) 를 thread pool 에서. event loop 보호.

---

## 9. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/file-upload-s3#9 함정 모음|§9]] + Flask 변형 함정)

### FastAPI 한정

### 함정 11 — `aioboto3.Session().client(...)` 매번 생성
overhead. **module-level session** + 매 호출 client (`async with`).

### 함정 12 — PIL `Image.open` async wrapping 안 하면 event loop block
큰 이미지 = 수 초 block. **`asyncio.to_thread`** 또는 Celery worker.

### 함정 13 — `BackgroundTasks` 가 response 후 실행
test 에서 task 결과 검증 어려움. **명시적 sync 처리** 또는 `dependency_overrides` 로 mock.

---

## 10. 운영 체크리스트

(Java 원본 §10 + Python async)

---

## 11. 관련

- [[../../spring/java-spring/api-design/file-upload-s3|↗ Java 원본]]
- [[../../python-flask/api-design/file-upload-s3|↗ Flask 변형]]
- [[../common/security-config]]
- [[api-design|↑ hub]]
