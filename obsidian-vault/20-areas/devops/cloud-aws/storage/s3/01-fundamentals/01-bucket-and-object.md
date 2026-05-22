---
title: "S3 01-1 — Bucket + Object + Key + Metadata + Version"
kind: knowledge
status: current
created_at: 2026-05-22T18:32:00+09:00
tags: [aws, s3, fundamentals, bucket, object, versioning, multipart]
home_hub: s3
---

# S3 Bucket + Object

> Bucket (글로벌 unique name) + Object (key + value + metadata + version). 5 TB object size limit, unlimited bucket size.

## 1. Bucket

```bash
aws s3 mb s3://bkurs-uploads-prod --region ap-northeast-2

# 또는 API
aws s3api create-bucket --bucket bkurs-uploads-prod \
  --region ap-northeast-2 \
  --create-bucket-configuration LocationConstraint=ap-northeast-2
```

### Bucket naming 규칙
- 3-63 char
- 소문자, 숫자, 하이픈, 점
- 시작/끝 = 영숫자
- IPv4 형식 안 됨
- 글로벌 unique
- 점 사용 시 — virtual-host URL 의 wildcard SSL 안 됨

### Bucket 의 region
- 생성 시 region 결정
- **이동 불가** — 새 bucket 으로 복사 필요
- Cross-region replication 으로 mirror

## 2. Object

```bash
# Upload
aws s3 cp report.pdf s3://bkurs-uploads-prod/reports/2026/05/report.pdf

# Download
aws s3 cp s3://bkurs-uploads-prod/reports/2026/05/report.pdf .

# List
aws s3 ls s3://bkurs-uploads-prod/reports/2026/ --recursive

# Delete
aws s3 rm s3://bkurs-uploads-prod/reports/2026/05/report.pdf

# Object metadata
aws s3api head-object --bucket bkurs-uploads-prod --key reports/2026/05/report.pdf
```

| 한도 | |
| --- | --- |
| Object size | 0 - 5 TB |
| Single PUT | up to 5 GB |
| Multipart Upload | up to 5 TB |
| Key length | 1024 chars |
| Metadata size | 2 KB |

## 3. Key 의 의미

```
s3://bkurs-uploads-prod/reports/2026/05/report.pdf
                       └────────────┬───────────┘
                                  Key
```

> S3 는 **flat namespace** — "directory" 는 prefix 의 시각화. 실제 directory 없음.

```bash
# 같은 prefix grouping
aws s3 ls s3://bkurs/reports/2026/         # 2026 의 모든 sub-prefix + object
aws s3 ls s3://bkurs/ --recursive          # 모든 object
```

### Key 설계 best practice
```
# 좋은 패턴
reports/2026/05/22/report-${id}.pdf
uploads/${userId}/avatar.jpg
logs/${env}/${service}/${date}/${hour}.log

# 나쁜 패턴 (모든 object 가 같은 prefix → throughput 한도)
data/2026-05-22-001
data/2026-05-22-002
...
```

> 옛날: prefix 다양화로 partition. 2018+ S3 가 자동 partition (필요 없음). 그러나 대량 list/scan 시 prefix 활용 유리.

## 4. Metadata

| 종류 | 의미 |
| --- | --- |
| **System metadata** | `Content-Type`, `Content-Length`, `Last-Modified`, `ETag` |
| **User metadata** | `x-amz-meta-*` (사용자 정의) |

```bash
aws s3 cp file.json s3://bkurs/ \
  --metadata "uploaded-by=alice,project=bkurs" \
  --content-type "application/json" \
  --cache-control "max-age=86400" \
  --content-disposition "attachment; filename=report.json"
```

> User metadata 는 immutable — copy 로 변경 가능.

## 5. Versioning

```bash
aws s3api put-bucket-versioning --bucket bkurs-uploads-prod \
  --versioning-configuration Status=Enabled
```

- Enable 후 모든 변경 = 새 version
- delete → delete marker 추가
- 이전 version restore 가능
- **disable 불가** — Suspended 만 (기존 version 유지)

```bash
# Versions list
aws s3api list-object-versions --bucket bkurs-uploads-prod --prefix file.txt

# 특정 version
aws s3api get-object --bucket bkurs-uploads-prod --key file.txt \
  --version-id "abc..." restored-file.txt

# Permanent delete (versionId 명시)
aws s3api delete-object --bucket bkurs-uploads-prod --key file.txt \
  --version-id abc...
```

## 6. ETag

```
ETag = MD5 hash (single upload, no encryption) 또는 multipart hash
```

| Case | ETag |
| --- | --- |
| Single PUT, no SSE-KMS | MD5 |
| Multipart | `<md5-of-md5s>-<part-count>` |
| SSE-KMS / SSE-C | 임의 (MD5 아님) |
| DSSE-KMS | 임의 |

→ checksum verification 시 ETag 대신 **Content-MD5 / SHA256** 권장.

## 7. Multipart Upload

```bash
# 100 MB 자동 multipart (aws cli)
aws s3 cp big-file.zip s3://bkurs/

# 강제 multipart
aws s3api create-multipart-upload --bucket bkurs --key big.zip
# returns UploadId

aws s3api upload-part --bucket bkurs --key big.zip \
  --part-number 1 --body part1.bin --upload-id $UPLOAD_ID

aws s3api complete-multipart-upload --bucket bkurs --key big.zip \
  --multipart-upload "Parts=[...]" --upload-id $UPLOAD_ID
```

### Multipart 한도
- Part size: 5 MB - 5 GB (last part < 5 MB OK)
- Part count: 1 - 10,000
- Object size: max 5 TB

### Multipart 의 장점
- Parallel upload (속도)
- Resumable (개별 part 재시도)
- 5 GB 초과 가능

### Aborted Multipart 정리
```bash
# 7일 후 자동 삭제 lifecycle
aws s3api put-bucket-lifecycle-configuration --bucket bkurs \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "abort-multipart",
      "Status": "Enabled",
      "Filter": {},
      "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 7 }
    }]
  }'
```

> 미완료 multipart 는 보이지 않지만 storage 비용 발생.

## 8. Checksum (2022+)

upload 시 algorithm 명시:
```bash
aws s3api put-object --bucket bkurs --key file.json --body file.json \
  --checksum-algorithm SHA256
```

→ S3 가 자동 계산 + storage 시 검증.

## 9. Object Tagging

```bash
aws s3api put-object-tagging --bucket bkurs --key file.pdf \
  --tagging 'TagSet=[{Key=Environment,Value=prod},{Key=Owner,Value=team-a}]'

# Lifecycle 의 Filter 에서 사용
"Filter": { "Tag": { "Key": "Environment", "Value": "dev" } }
```

→ Lifecycle / Replication / IAM Condition / Cost Explorer.

## 10. Object Hold

[[../04-security/02-object-lock-and-mfa-delete]]

## 11. Storage Lens

```bash
aws s3control put-storage-lens-configuration \
  --account-id $ACCT --config-id default \
  --storage-lens-configuration '{ "AccountLevel": {...}, "IsEnabled": true }'
```

→ account-wide visibility (object count, storage, request, lifecycle).

## 12. CLI vs SDK

| 도구 | 용도 |
| --- | --- |
| `aws s3 cp/sync/ls` | 빠른 사용 (auto multipart) |
| `aws s3api ...` | 세밀한 control |
| SDK (boto3 / Java SDK v2) | code 안 |
| s3fs / mountpoint-s3 | filesystem mount |
| CloudFront | CDN |
| AWS DataSync | bulk migration |

## 13. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Bucket name 글로벌 unique 모름 | unique 명명 |
| Region 이동 못 함 | 처음 결정 |
| Versioning 활성 후 disable 안 됨 | Suspended 만 가능 |
| ETag = MD5 라고 가정 (Multipart 아님) | checksum 별도 |
| Multipart aborted 정리 안 함 | lifecycle abort |
| Key prefix 다양화 안 함 (옛 권고) | 2018+ 불필요 |
| User metadata 수정 시도 | copy 로만 |

## 14. 다음

- ▶ [[02-storage-classes]]
- ▶ [[03-pricing-and-limits]]
- ▶ [[../02-access/01-bucket-policy-and-acl]]

## 15. 영상 / 자료

| 자료 |
| --- |
| [S3 공식](https://docs.aws.amazon.com/AmazonS3/latest/userguide/) |
| [Multipart Upload](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html) |
| [Storage Lens](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage_lens.html) |

## 16. 참고

- [[../s3]]
- [[../../storage]]
