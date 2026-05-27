---
title: "S3 02-3 — Presigned URL + CORS"
kind: knowledge
status: current
created_at: 2026-05-22T18:44:00+09:00
tags: [aws, s3, access, presigned, cors, upload]
home_hub: s3
---

# Presigned URL + CORS

> 임시 권한 (default 1h, max 7d) — browser 직접 upload/download. CORS = cross-origin 허용.

## 1. Presigned URL 의 의미

```
S3 owner 의 IAM 권한을 짧은 URL 에 위임
→ URL 받은 사람은 IAM 없이도 S3 접근 가능
→ 만료 시간 후 거부
```

## 2. Presigned GET (download)

```python
import boto3
s3 = boto3.client("s3")
url = s3.generate_presigned_url(
    "get_object",
    Params={"Bucket": "bkurs", "Key": "file.pdf"},
    ExpiresIn=3600    # 1h
)
```

```bash
aws s3 presign s3://bkurs/file.pdf --expires-in 3600
```

→ URL 받은 사람이 1h 안 download.

## 3. Presigned PUT (upload)

```python
url = s3.generate_presigned_url(
    "put_object",
    Params={
        "Bucket": "bkurs-uploads",
        "Key": "uploads/${user_id}/${file_id}",
        "ContentType": "image/jpeg",
        "Metadata": {"user-id": "u-abc"}
    },
    ExpiresIn=900     # 15분
)
```

Client (browser):
```javascript
const file = document.querySelector('input[type="file"]').files[0];
await fetch(url, {
  method: 'PUT',
  body: file,
  headers: { 'Content-Type': 'image/jpeg' }
});
```

> Server 안 거치고 browser → S3 직접. server load ↓.

## 4. Presigned POST (form upload)

PUT 보다 더 풍부한 제어 (size limit / metadata 등):

```python
import boto3
s3 = boto3.client("s3")

post = s3.generate_presigned_post(
    Bucket="bkurs-uploads",
    Key="uploads/${filename}",
    Fields={"acl": "private", "Content-Type": "image/jpeg"},
    Conditions=[
        {"acl": "private"},
        {"Content-Type": "image/jpeg"},
        ["content-length-range", 0, 5*1024*1024],   # max 5 MB
        ["starts-with", "$key", "uploads/"]
    ],
    ExpiresIn=900
)
# post = { "url": "...", "fields": { ... } }
```

Client:
```html
<form action="{{ post.url }}" method="POST" enctype="multipart/form-data">
  <input type="hidden" name="key" value="uploads/cat.jpg" />
  <input type="hidden" name="X-Amz-Credential" value="{{ post.fields['X-Amz-Credential'] }}" />
  <!-- 모든 fields -->
  <input type="file" name="file" />
  <button>Upload</button>
</form>
```

## 5. 만료 시간

| | |
| --- | --- |
| Default | 15분 (CLI) / 1h (SDK) |
| Maximum | **7일** |
| Minimum | 1초 |
| 임시 credential (Role) | session 만료 시 |
| IAM user signature | 7일 |

> Role 의 sessionDuration 이 짧으면 그게 cap.

## 6. Multipart Presigned

각 part 마다 presigned URL:
```python
upload = s3.create_multipart_upload(Bucket="bkurs", Key="big.zip")
upload_id = upload["UploadId"]

urls = []
for part_number in range(1, total_parts+1):
    url = s3.generate_presigned_url(
        "upload_part",
        Params={
            "Bucket": "bkurs",
            "Key": "big.zip",
            "UploadId": upload_id,
            "PartNumber": part_number
        },
        ExpiresIn=3600
    )
    urls.append({"part": part_number, "url": url})

# client 가 각 part upload 후
# server 가 complete_multipart_upload
```

## 7. CORS — Cross-Origin Resource Sharing

Browser 의 다른 origin (예: bkurs.com → bkurs-uploads.s3.amazonaws.com) 호출 시:
```bash
aws s3api put-bucket-cors --bucket bkurs-uploads \
  --cors-configuration '{
    "CORSRules": [{
      "AllowedHeaders": ["*"],
      "AllowedMethods": ["GET","PUT","POST","DELETE","HEAD"],
      "AllowedOrigins": ["https://bkurs.com", "https://www.bkurs.com"],
      "ExposeHeaders": ["ETag", "x-amz-version-id"],
      "MaxAgeSeconds": 3000
    }]
  }'
```

| Field | 의미 |
| --- | --- |
| `AllowedOrigins` | 호출 가능한 도메인 |
| `AllowedMethods` | GET / PUT / POST / DELETE / HEAD |
| `AllowedHeaders` | request header (preflight 응답) |
| `ExposeHeaders` | response 에 노출할 header |
| `MaxAgeSeconds` | preflight cache 시간 |

## 8. CORS 의 preflight

```
1. Browser → OPTIONS /file.json (Origin: https://bkurs.com)
2. S3 → 200 + Access-Control-Allow-Origin header
3. Browser → 실제 PUT
```

→ MaxAgeSeconds 동안 preflight cache.

## 9. Presigned URL + CORS 결합

```
Server:        Presigned URL 발급
Browser:       URL 로 직접 PUT (Origin: https://bkurs.com)
S3:            CORS check + signature check → OK
```

> **둘 다 필요** — Presigned 가 권한 / CORS 가 cross-origin.

## 10. 보안 best practice

### 1) 짧은 만료
```python
ExpiresIn=900   # 15분 권장
```

### 2) Size 제한 (Presigned POST)
```python
Conditions=[
    ["content-length-range", 0, 5*1024*1024]   # 5 MB max
]
```

### 3) Content-Type 강제
```python
Conditions=[
    {"Content-Type": "image/jpeg"}
]
```

### 4) Key prefix 강제
```python
Conditions=[
    ["starts-with", "$key", f"uploads/{user_id}/"]
]
```

### 5) MIME type 검증 (server 추가)
```python
# upload 후 Lambda 에서 magic byte 검증
```

## 11. Cross-region Presigned

```python
import boto3
# us-east-1 의 bucket → ap-northeast-2 client 도 호출 가능 (signature region 일치)
s3 = boto3.client("s3", region_name="us-east-1",
                  config=boto3.session.Config(signature_version="s3v4"))
url = s3.generate_presigned_url(...)
```

## 12. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| 만료 너무 김 (7일) | 15분 |
| Size 제한 없음 → 큰 파일 abuse | content-length-range |
| Content-Type 강제 안 함 | 명시 |
| Key 패턴 강제 안 함 | starts-with |
| CORS 안 함 → browser 차단 | bucket CORS |
| ExpiresIn > Role 의 sessionDuration | role 의 cap |
| Signature region 잘못 | client region == bucket region |

## 13. 활용 패턴

### 1) Browser direct upload
```
Browser → /api/upload-url (서버)
Server → IAM check → presigned PUT URL 반환
Browser → S3 직접 PUT
Browser → /api/upload-complete (서버 알림)
Server → DB 에 metadata 저장
```

### 2) Email/SMS 의 download link
```
서버 → presigned GET URL (1h)
이메일 본문에 URL
```

### 3) S3 의 임시 sharing
```
aws s3 presign s3://bkurs/file.pdf --expires-in 86400
```

## 14. 다음

- ▶ [[../03-lifecycle/01-lifecycle-policy-and-tiering]]
- ▶ [[../04-security/01-encryption-sse-kms-c-bucket-keys]]
- ▶ [[../practice/02-presigned-upload-pipeline]]

## 15. 영상 / 자료

| 자료 |
| --- |
| [Presigned URL](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html) |
| [Presigned POST](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/s3-presigned-urls.html) |
| [CORS 공식](https://docs.aws.amazon.com/AmazonS3/latest/userguide/cors.html) |

## 16. 참고

- [[../s3]]
- [[01-bucket-policy-and-acl]]
- [[02-block-public-access-and-vpc-endpoint]]
