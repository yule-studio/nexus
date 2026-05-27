---
title: "S3 03-1 — Lifecycle Policy + Intelligent-Tiering"
kind: knowledge
status: current
created_at: 2026-05-22T18:50:00+09:00
tags: [aws, s3, lifecycle, intelligent-tiering, glacier, transition]
home_hub: s3
---

# S3 Lifecycle Policy + Intelligent-Tiering

> 시간 기반 자동 transition / expire. Standard → IA → Glacier → 삭제. Intelligent-Tiering 은 access 기반 자동.

## 1. Lifecycle Rule 구조

```json
{
  "Rules": [{
    "ID": "archive-old-reports",
    "Status": "Enabled",
    "Filter": { "Prefix": "reports/" },
    "Transitions": [
      { "Days": 30,  "StorageClass": "STANDARD_IA" },
      { "Days": 90,  "StorageClass": "GLACIER" },
      { "Days": 365, "StorageClass": "DEEP_ARCHIVE" }
    ],
    "Expiration": { "Days": 2555 },
    "NoncurrentVersionTransitions": [
      { "NoncurrentDays": 30, "StorageClass": "GLACIER" }
    ],
    "NoncurrentVersionExpiration": { "NoncurrentDays": 90 },
    "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 7 },
    "ExpiredObjectDeleteMarker": true
  }]
}
```

## 2. 적용

```bash
aws s3api put-bucket-lifecycle-configuration --bucket bkurs \
  --lifecycle-configuration file://lifecycle.json
```

## 3. 5 종류 action

| Action | 의미 |
| --- | --- |
| `Transitions` | current version 의 storage class 이동 |
| `Expiration` | current version 삭제 (versioning 시 delete marker) |
| `NoncurrentVersionTransitions` | old version 의 transition |
| `NoncurrentVersionExpiration` | old version permanent delete |
| `AbortIncompleteMultipartUpload` | 미완료 multipart 정리 |
| `ExpiredObjectDeleteMarker` | versioning 의 delete marker 정리 |

## 4. Filter 옵션

```json
"Filter": {
  "Prefix": "logs/"
}

"Filter": {
  "Tag": { "Key": "Environment", "Value": "dev" }
}

"Filter": {
  "And": {
    "Prefix": "logs/",
    "Tags": [{ "Key": "Project", "Value": "bkurs" }],
    "ObjectSizeGreaterThan": 1048576,
    "ObjectSizeLessThan": 5368709120
  }
}
```

## 5. Transition 규칙

```
Source → Target 의 제약:
- Standard / Reduced Redundancy → IA / One Zone-IA / Glacier 3종
- IA → One Zone-IA / Glacier 3종
- Glacier IR → Glacier Flexible / Deep
- Glacier Flexible → Deep

Min duration:
- IA: 30일 (30일 안에 transition X)
- Glacier: 30일 (Standard / IA → Glacier 가능)
```

## 6. 흔한 패턴

### Pattern A — 일반 데이터
```json
{
  "Transitions": [
    { "Days": 30,  "StorageClass": "STANDARD_IA" },
    { "Days": 90,  "StorageClass": "GLACIER" },
    { "Days": 365, "StorageClass": "DEEP_ARCHIVE" }
  ],
  "Expiration": { "Days": 2555 }   // 7년
}
```

### Pattern B — 로그 (자주 보지 않음)
```json
{
  "Transitions": [
    { "Days": 1,   "StorageClass": "STANDARD_IA" },
    { "Days": 30,  "StorageClass": "GLACIER" }
  ],
  "Expiration": { "Days": 365 }
}
```

### Pattern C — 임시 (build artifact)
```json
{
  "Filter": { "Prefix": "build/" },
  "Expiration": { "Days": 30 }
}
```

### Pattern D — versioning 정리
```json
{
  "NoncurrentVersionTransitions": [
    { "NoncurrentDays": 30, "StorageClass": "GLACIER" }
  ],
  "NoncurrentVersionExpiration": { "NoncurrentDays": 90 }
}
```

### Pattern E — Multipart 정리 (모든 bucket 권장)
```json
{
  "ID": "abort-multipart",
  "Status": "Enabled",
  "Filter": {},
  "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 7 }
}
```

## 7. Intelligent-Tiering — 자동

```bash
aws s3 cp file.json s3://bkurs/data/ --storage-class INTELLIGENT_TIERING
```

```
30일 미접근  → Infrequent Access tier
90일 미접근  → Archive Instant Access
180일 미접근 → Archive Access (optional)
365일 미접근 → Deep Archive Access (optional)
```

설정:
```bash
aws s3api put-bucket-intelligent-tiering-configuration --bucket bkurs \
  --id default \
  --intelligent-tiering-configuration '{
    "Id": "default",
    "Status": "Enabled",
    "Tierings": [
      { "Days": 180, "AccessTier": "ARCHIVE_ACCESS" },
      { "Days": 365, "AccessTier": "DEEP_ARCHIVE_ACCESS" }
    ]
  }'
```

> **modern default** — access pattern 모를 때 권장.

## 8. Transition 비용

```
$0.011 / 1K transition request
+ 모든 lifecycle 이 24시간마다 evaluate (무료)
```

큰 file 의 transition = 거의 무료. 작은 file 대량 = 비용 발생.

## 9. Dry-run

```bash
# console 또는 SDK 의 simulate (S3 Storage Lens 활용)
aws s3control get-storage-lens-configuration --account-id $ACCT --config-id default
```

→ object distribution 보고 lifecycle 결정.

## 10. CloudWatch event — Lifecycle action

```yaml
EventPattern:
  source: [aws.s3]
  detail-type: ["Object Storage Class Changed"]
  detail:
    bucket: { name: ["bkurs-uploads"] }
```

→ Lifecycle 실행 시 audit / notification.

## 11. CloudTrail — Lifecycle config 변경

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=PutBucketLifecycle \
  --max-items 20
```

## 12. 다국 적용

```bash
# Multi-bucket 일괄
for bucket in bkurs-prod bkurs-staging bkurs-dev; do
  aws s3api put-bucket-lifecycle-configuration --bucket $bucket \
    --lifecycle-configuration file://lifecycle.json
done
```

## 13. ExpiredObjectDeleteMarker

versioning + lifecycle expire 시:
```
1. Lifecycle expire current version → delete marker 추가
2. Old version 만 남음 + delete marker
3. ExpiredObjectDeleteMarker=true → 모든 noncurrent expire 후 delete marker 자동 정리
```

→ versioning bucket 의 storage 가 무한히 늘어남 방지.

## 14. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Lifecycle 없음 → storage 폭주 | 항상 설정 |
| Versioning 의 old version expire 안 함 | NoncurrentVersionExpiration |
| Delete marker 누적 | ExpiredObjectDeleteMarker |
| Multipart aborted 보이지 않음 | AbortIncompleteMultipartUpload 7d |
| Filter prefix typo | console 확인 |
| Min duration 인지 못 함 (IA 30d) | days >= 30 |
| Intelligent-Tiering 작은 file 비효율 | 128 KB+ 만 |

## 15. 다음

- ▶ [[02-replication-and-batch-ops]]
- ▶ [[03-event-notifications-and-eventbridge]]
- ▶ [[../practice/03-lifecycle-glacier-deep-archive]]

## 16. 영상 / 자료

| 자료 |
| --- |
| [Lifecycle 공식](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html) |
| [Intelligent-Tiering](https://aws.amazon.com/s3/storage-classes/intelligent-tiering/) |

## 17. 참고

- [[../s3]]
- [[../01-fundamentals/02-storage-classes]]
