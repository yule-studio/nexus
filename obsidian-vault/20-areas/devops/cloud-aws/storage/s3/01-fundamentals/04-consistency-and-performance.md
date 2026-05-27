---
title: "S3 01-4 — Consistency + Performance (Strong / Multipart / Transfer Accel)"
kind: knowledge
status: current
created_at: 2026-05-22T18:38:00+09:00
tags: [aws, s3, fundamentals, consistency, performance, multipart, transfer-acceleration]
home_hub: s3
---

# S3 Consistency + Performance

> 2020+ **Strong Consistency** (read-after-write) for all. Multipart / Transfer Acceleration / Byte Range / S3 Select / S3 Express.

## 1. Strong Consistency (2020+)

```
PUT → 즉시 GET 으로 보임 (모든 region)
LIST 도 즉시 일관
overwrite + DELETE 도 즉시 일관
```

→ 이전 (eventually consistent for overwrite/delete) 사라짐. 모든 application 이 strong.

```python
s3.put_object(Bucket="bkurs", Key="file.json", Body=data)
# 즉시
s3.get_object(Bucket="bkurs", Key="file.json")   # 100% 새 data
```

## 2. Request Rate

```
Per prefix:
- GET: 5500 RPS
- PUT/POST/DELETE: 3500 RPS

자동 partition — 다양한 prefix = 거의 무한.
```

옛 권고 (prefix randomization) — 2018+ 불필요. 단 매우 큰 throughput 시 prefix 다양화 도움.

## 3. Multipart Upload

> 5 MB 이상 강력 권장, 100 MB+ 필수.

```bash
# CLI 자동 — multipart_threshold + multipart_chunksize
aws configure set default.s3.multipart_threshold 64MB
aws configure set default.s3.multipart_chunksize 16MB
aws configure set default.s3.max_concurrent_requests 20

aws s3 cp big.zip s3://bkurs/
```

### 장점
- Parallel upload (대역폭 활용)
- Resumable (part 단위 재시도)
- 5 GB 초과 가능

### Boto3 자동
```python
import boto3
s3 = boto3.client("s3")
# 큰 file 자동 multipart
s3.upload_file("big.zip", "bkurs", "big.zip")

# 또는 직접
from boto3.s3.transfer import TransferConfig
config = TransferConfig(
    multipart_threshold=64*1024*1024,
    max_concurrency=20,
    multipart_chunksize=16*1024*1024,
)
s3.upload_file("big.zip", "bkurs", "big.zip", Config=config)
```

자세히 → [[01-bucket-and-object]] §7

## 4. Byte Range GET

```bash
aws s3api get-object --bucket bkurs --key big.zip \
  --range "bytes=0-1048575" part1.bin

aws s3api get-object --bucket bkurs --key big.zip \
  --range "bytes=1048576-2097151" part2.bin
```

→ 큰 file 의 일부만 download. parallel resume.

## 5. Transfer Acceleration

```bash
aws s3api put-bucket-accelerate-configuration --bucket bkurs \
  --accelerate-configuration Status=Enabled

# URL 변경
aws s3 cp big.zip s3://bkurs.s3-accelerate.amazonaws.com/big.zip
```

→ CloudFront edge 경유 후 S3 로 upload. 먼 거리 (예: 한국 → us-east-1) 시 10-500% 빠름.

| 가격 |
| --- |
| $0.04/GB premium (Asia → US) |

언제:
- Cross-continent upload
- Mobile user → 먼 region

## 6. S3 Select / Glacier Select

```bash
aws s3api select-object-content --bucket bkurs --key data.csv \
  --expression "SELECT s.name, s.age FROM s3object s WHERE s.age > 30" \
  --expression-type SQL \
  --input-serialization '{"CSV":{"FileHeaderInfo":"USE"}}' \
  --output-serialization '{"CSV":{}}' \
  selected.csv
```

→ object 전체 download 안 하고 SQL filter. CSV / JSON / Parquet 지원.

> 비용 절감 — full GET 안 함. Athena 의 더 강력한 alternative.

## 7. Glacier Select

Glacier 안의 object 도 같은 방식 (restore 안 하고).

## 8. S3 Express One Zone

```bash
aws s3api create-bucket --bucket bkurs-express--use1-az1--x-s3 \
  --create-bucket-configuration '{
    "Location": { "Type": "AvailabilityZone", "Name": "use1-az1" },
    "Bucket": { "DataRedundancy": "SingleAvailabilityZone", "Type": "Directory" }
  }'
```

- single-AZ
- μs-ms latency (10x ↑)
- 50% 싸진 request
- session-based auth (10000 req/s per token)
- Mountpoint-S3 / CSI Driver 통합

활용:
- ML training (dataset random access)
- Analytics shuffle
- IoT 짧은 hot tier
- Container ephemeral

## 9. Mountpoint for Amazon S3

POSIX-like filesystem:
```bash
mkdir /mnt/s3
mount-s3 bkurs-uploads /mnt/s3 \
  --region ap-northeast-2 \
  --cache /tmp/s3-cache \
  --max-cache-size-mb 5000

ls /mnt/s3/
cat /mnt/s3/file.json
```

→ S3 를 filesystem 처럼. write 는 sequential only (random write ✗).

## 10. Performance Patterns

### 큰 download
```python
# Parallel byte range
def parallel_download(bucket, key, dest, parts=10):
    size = s3.head_object(Bucket=bucket, Key=key)["ContentLength"]
    chunk = size // parts
    with ThreadPoolExecutor(max_workers=parts) as exec:
        for i in range(parts):
            start = i * chunk
            end = start + chunk - 1 if i < parts-1 else size - 1
            exec.submit(download_range, bucket, key, start, end, f"{dest}.{i}")
```

### 큰 upload
```python
# TransferManager 자동 multipart + parallel
s3.upload_file("big.zip", "bkurs", "big.zip", Config=multipart_config)
```

### List 대량
```python
# Pagination (5500/s 한도 안)
paginator = s3.get_paginator("list_objects_v2")
for page in paginator.paginate(Bucket="bkurs", Prefix="reports/"):
    for obj in page.get("Contents", []):
        print(obj["Key"])
```

> S3 Inventory 가 큰 list 작업에 더 효율적.

## 11. S3 Inventory

```bash
aws s3api put-bucket-inventory-configuration --bucket bkurs \
  --id daily-inventory --inventory-configuration '{
    "Id": "daily-inventory",
    "IsEnabled": true,
    "Destination": {
      "S3BucketDestination": {
        "Bucket": "arn:aws:s3:::bkurs-inventory",
        "Format": "CSV"
      }
    },
    "Schedule": { "Frequency": "Daily" },
    "IncludedObjectVersions": "Current",
    "OptionalFields": ["Size","StorageClass","LastModifiedDate","ETag","EncryptionStatus"]
  }'
```

→ 매일 CSV/Parquet 으로 bucket 의 object list. Athena 로 query 가능.

## 12. 동시 access 의 한도

```
S3 가 자동 scale.
Hot prefix 가 너무 빠르게 늘면 503 Slow Down → backoff + 재시도.
```

> SDK 가 자동 retry — 대부분 인지 못 함.

## 13. CloudWatch metric

```
BucketSizeBytes        / NumberOfObjects   (daily)
AllRequests            (1-min, 활성)
GetRequests / PutRequests / DeleteRequests
4xxErrors / 5xxErrors
FirstByteLatency / TotalRequestLatency
ReplicationLatency
```

→ Bucket Request Metric 활성 ($0.30/M req).

## 14. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| 큰 file single PUT (5 GB+) | multipart |
| ETag = MD5 가정 (multipart 아님) | checksum 별도 |
| Strong consistency 안 활용 | 즉시 GET OK |
| Prefix 1 곳에 모두 (hot) | naturally diverse |
| Inventory 없음 → 큰 list 비용 | Inventory daily |
| Transfer Acceleration 한국 내 사용 | cross-continent 만 |
| Mountpoint random write | sequential 만 |

## 15. 다음

- ▶ [[../02-access/01-bucket-policy-and-acl]]
- ▶ [[../03-lifecycle/01-lifecycle-policy-and-tiering]]
- ▶ [[../04-security/01-encryption-sse-kms-c-bucket-keys]]

## 16. 영상 / 자료

| 자료 |
| --- |
| [S3 Performance — re:Invent](https://www.youtube.com/watch?v=GDQAWBpFnUM) |
| [Strong Consistency 발표](https://aws.amazon.com/blogs/aws/amazon-s3-update-strong-read-after-write-consistency/) |
| [Mountpoint-S3](https://github.com/awslabs/mountpoint-s3) |

## 17. 참고

- [[../s3]]
- [[01-bucket-and-object]]
- [[02-storage-classes]]
