---
title: "파일/이미지 업로드 (S3 presigned URL) — Flask"
kind: knowledge
status: current
created_at: 2026-05-26T17:50:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-flask, api-design, recipe, file-upload, s3]
---

# 파일/이미지 업로드 (S3 presigned URL) — Flask

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/file-upload-s3|↗ Java 원본]]**

> `boto3` + S3 presigned URL 패턴.

---

## 1-2. 왜 presigned / 4. CORS

(Java 원본 [[../../spring/java-spring/api-design/file-upload-s3#1|§1, §7]] 와 동일)

---

## 3. 도메인 + DB

```python
from enum import Enum
from sqlalchemy import Column, String, BigInteger, DateTime
from datetime import datetime, timezone


class FileKind(Enum):
    PROFILE_IMAGE = "PROFILE_IMAGE"
    PRODUCT_IMAGE = "PRODUCT_IMAGE"
    REVIEW_IMAGE = "REVIEW_IMAGE"
    ATTACHMENT = "ATTACHMENT"
    VIDEO = "VIDEO"


class FileStatus(Enum):
    PENDING = "PENDING"
    COMPLETED = "COMPLETED"
    FAILED = "FAILED"
    DELETED = "DELETED"


class FileMeta(Base):
    __tablename__ = "files"
    id = Column(String(26), primary_key=True)
    owner_id = Column(String(26), nullable=False)
    kind = Column(String(30), nullable=False)
    s3_bucket = Column(String(100), nullable=False)
    s3_key = Column(String(500), nullable=False)
    content_type = Column(String(100), nullable=False)
    size_bytes = Column(BigInteger, nullable=False)
    original_name = Column(String(255))
    status = Column(String(20), nullable=False)
    created_at = Column(DateTime(timezone=True), default=lambda: datetime.now(timezone.utc))
    completed_at = Column(DateTime(timezone=True))
```

---

## 4. S3 Service — boto3 presigner

```python
import boto3
from botocore.exceptions import ClientError
from datetime import datetime, timedelta, timezone

s3_client = boto3.client("s3", region_name="ap-northeast-2")


class S3FileService:
    def __init__(self, bucket: str, upload_ttl_min: int = 5, download_ttl_min: int = 15,
                 cdn_base_url: str | None = None) -> None:
        self.bucket = bucket
        self.upload_ttl = upload_ttl_min * 60
        self.download_ttl = download_ttl_min * 60
        self.cdn_base_url = cdn_base_url

    def build_upload_url(self, key: str, content_type: str, size_bytes: int) -> dict:
        url = s3_client.generate_presigned_url(
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

    def build_download_url(self, key: str) -> str:
        if self.cdn_base_url:
            return f"{self.cdn_base_url}/{key}"
        return s3_client.generate_presigned_url(
            "get_object",
            Params={"Bucket": self.bucket, "Key": key},
            ExpiresIn=self.download_ttl,
        )

    def verify(self, key: str) -> dict:
        try:
            return s3_client.head_object(Bucket=self.bucket, Key=key)
        except ClientError as e:
            if e.response["Error"]["Code"] == "404":
                raise BusinessException(ResponseCode.NOT_FOUND, "S3 에 파일이 없습니다.")
            raise
```

---

## 5. UseCase

```python
ALLOWED_IMAGE_TYPES = {"image/jpeg", "image/png", "image/webp"}
ALLOWED_VIDEO_TYPES = {"video/mp4", "video/quicktime"}
MAX_IMAGE_BYTES = 10 * 1024 * 1024
MAX_VIDEO_BYTES = 500 * 1024 * 1024


def issue_upload_url(owner_id: str, kind: FileKind, content_type: str,
                     size_bytes: int, original_filename: str | None) -> dict:
    _validate(kind, content_type, size_bytes)

    file_id = ulid_gen()
    key = _build_key(kind, owner_id, file_id, original_filename)
    presigned = s3_service.build_upload_url(key, content_type, size_bytes)

    db.session.add(FileMeta(
        id=file_id, owner_id=owner_id, kind=kind.value,
        s3_bucket=presigned["bucket"], s3_key=presigned["key"],
        content_type=content_type, size_bytes=size_bytes,
        original_name=original_filename, status=FileStatus.PENDING.value,
    ))
    db.session.commit()

    return {"fileId": file_id, **presigned}


def complete_upload(file_id: str, current_user: str) -> None:
    meta = file_repo.find_by_id(file_id)
    if meta is None:
        raise BusinessException(ResponseCode.NOT_FOUND, "파일")
    if meta.owner_id != current_user:
        raise BusinessException(ResponseCode.FORBIDDEN)
    if meta.status != FileStatus.PENDING.value:
        raise BusinessException(ResponseCode.CONFLICT, "이미 완료")

    head = s3_service.verify(meta.s3_key)
    if head["ContentLength"] != meta.size_bytes:
        raise BusinessException(ResponseCode.BAD_REQUEST, "크기 불일치")

    meta.status = FileStatus.COMPLETED.value
    meta.completed_at = datetime.now(timezone.utc)
    db.session.commit()

    # 후처리 trigger — Celery
    process_uploaded_file.delay(file_id)


def _validate(kind: FileKind, content_type: str, size_bytes: int) -> None:
    if size_bytes <= 0:
        raise BusinessException(ResponseCode.BAD_REQUEST, "유효하지 않은 파일 크기")

    if kind in (FileKind.PROFILE_IMAGE, FileKind.PRODUCT_IMAGE, FileKind.REVIEW_IMAGE):
        if content_type not in ALLOWED_IMAGE_TYPES:
            raise BusinessException(ResponseCode.BAD_REQUEST, "허용되지 않는 이미지 형식")
        if size_bytes > MAX_IMAGE_BYTES:
            raise BusinessException(ResponseCode.BAD_REQUEST, "이미지는 최대 10MB")
    elif kind == FileKind.VIDEO:
        if content_type not in ALLOWED_VIDEO_TYPES:
            raise BusinessException(ResponseCode.BAD_REQUEST, "허용되지 않는 동영상 형식")
        if size_bytes > MAX_VIDEO_BYTES:
            raise BusinessException(ResponseCode.BAD_REQUEST, "동영상은 최대 500MB")
```

---

## 6. Controller

```python
@file_bp.post("/presigned-url")
@jwt_required()
def presigned():
    payload = PresignedSchema().load(request.get_json())
    result = issue_upload_url(
        get_jwt_identity(),
        FileKind[payload["kind"]],
        payload["contentType"],
        payload["sizeBytes"],
        payload.get("originalFilename"),
    )
    return ok("URL 발급", result)


@file_bp.post("/<file_id>/complete")
@jwt_required()
def complete(file_id: str):
    complete_upload(file_id, get_jwt_identity())
    return ok("업로드 완료")
```

---

## 8. 후처리 — Celery + PIL

```python
from PIL import Image
import io


@shared_task(bind=True, max_retries=3)
def process_uploaded_file(self, file_id: str) -> None:
    meta = file_repo.find_by_id(file_id)
    if meta.kind != FileKind.PRODUCT_IMAGE.value:
        return

    try:
        obj = s3_client.get_object(Bucket=meta.s3_bucket, Key=meta.s3_key)
        img_bytes = obj["Body"].read()

        for size in (256, 512, 1024):
            img = Image.open(io.BytesIO(img_bytes))
            img.thumbnail((size, size))
            buf = io.BytesIO()
            img.save(buf, format="WEBP", quality=85)
            buf.seek(0)

            thumb_key = meta.s3_key.replace(meta.s3_key.split("/")[0], f"thumbnails/{size}", 1)
            s3_client.put_object(
                Bucket=meta.s3_bucket, Key=thumb_key,
                Body=buf.getvalue(), ContentType="image/webp",
            )
    except Exception as e:
        log.error("thumbnail failed for %s: %s", meta.s3_key, e)
        self.retry(countdown=60)
```

---

## 9. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/file-upload-s3#9 함정 모음|§9]] 의 11 함정 그대로)

### Python 한정

### 함정 12 — `boto3.client("s3")` global vs thread-local
`boto3` client 는 thread-safe (botocore 가 lock). 그러나 multi-process (gunicorn fork) 면 fork 후 connection 재초기화 권장 — `boto3.Session().client(...)`.

### 함정 13 — PIL `Image.open` 메모리 폭발
큰 이미지를 메모리에 통째로. `Image.open(stream).draft("RGB", (size, size))` 로 lazy 또는 size limit.

---

## 10. 운영 체크리스트

(Java 원본 §10 그대로)

---

## 11. 관련

- [[../../spring/java-spring/api-design/file-upload-s3|↗ Java 원본]]
- [[../common/security-config]]
- [[api-design|↑ hub]]
