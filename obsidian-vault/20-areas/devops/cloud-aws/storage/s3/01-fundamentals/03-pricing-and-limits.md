---
title: "S3 01-3 — Pricing + Limits"
kind: knowledge
status: current
created_at: 2026-05-22T18:36:00+09:00
tags: [aws, s3, fundamentals, pricing, limits]
home_hub: s3
---

# S3 Pricing + Limits

> Storage + Request + Transfer + Management features. Hard / soft limits.

## 1. 가격 구성

```
Storage (GB·월)
+ Request (PUT/GET/LIST count)
+ Transfer (egress to internet / cross-region)
+ Management (Inventory / Storage Lens / Object Tagging)
+ Replication / Storage Class transition
```

## 2. Storage cost (Seoul ap-northeast-2)

| Class | $/GB·월 |
| --- | --- |
| Standard | $0.025 |
| Standard-IA | $0.0138 |
| One Zone-IA | $0.011 |
| Glacier Instant Retrieval | $0.005 |
| Glacier Flexible Retrieval | $0.0045 |
| Glacier Deep Archive | $0.002 |
| S3 Express One Zone | $0.16 |

자세히 → [[02-storage-classes]]

## 3. Request cost

| Operation | Standard $/1K req |
| --- | --- |
| PUT / POST / LIST / COPY | $0.0054 |
| GET / SELECT | $0.00043 |
| Delete | 무료 |
| Lifecycle transition | $0.011/1K (Standard → IA / Glacier 이동) |

| Glacier retrieval |
| --- |
| Expedited: $0.011/1K + $0.03/GB |
| Standard: $0.011/1K + $0.01/GB |
| Bulk: $0.025/1K + $0.0025/GB |

> Small file 대량 = request cost 가 storage 보다 큼.

## 4. Data Transfer

| 방향 | 비용 |
| --- | --- |
| Internet → S3 (IN) | 무료 |
| S3 → Internet (OUT) | $0.114/GB (under 10 TB), 거리별 |
| S3 → CloudFront | 무료 |
| S3 → same region (EC2 / Lambda) | 무료 |
| S3 → other region | $0.09/GB |
| S3 cross-region replication | $0.09/GB (one-time) |
| Transfer Acceleration | $0.04/GB premium |

## 5. CloudFront 와 결합 시 비용

```
S3 → CloudFront (origin pull): 무료
CloudFront → Internet:        $0.114/GB → $0.085/GB (CF discount)
```

→ Internet egress 다량 시 CloudFront 가 더 싸다.

## 6. Hard limits

| 항목 | Limit |
| --- | --- |
| Bucket / account | 100 (soft, max 1000) |
| Object size | 5 TB |
| Single PUT | 5 GB |
| Multipart parts | 10000 |
| Min part | 5 MB (마지막 제외) |
| Key length | 1024 chars |
| Metadata (user) | 2 KB total |
| Lifecycle rules / bucket | 1000 |
| CORS rules / bucket | 100 |
| Bucket policy | 20 KB |
| Notification configurations | 100 |
| Tags / bucket | 50 |
| Tags / object | 10 |
| Replication rules | 1000 |

## 7. Soft limits (증액)

| | |
| --- | --- |
| Bucket / account | 100 → 1000 |
| GET request rate | 5,500/s per prefix (auto scale) |
| PUT/POST/DELETE | 3,500/s per prefix |

> Modern S3 (2018+) 는 prefix 별 자동 partition. Hot prefix 자체가 빠르게 scale.

## 8. Request rate — 최적화

```
초당 5500 GET / 3500 PUT per prefix
→ 다양한 prefix 사용 시 multiplier
```

예: 100 prefix × 5500 GET/s = 550K GET/s.

> 2018+ S3 가 자동 partition — prefix 다양화 부담 ↓. 다만 매우 큰 throughput 시 고려.

## 9. Free Tier (계정 12개월)

| | 무료 |
| --- | --- |
| Storage | 5 GB Standard |
| GET | 20,000 / 월 |
| PUT | 2,000 / 월 |
| DT outbound | 100 GB / 월 (모든 service 합산) |

## 10. 비용 시뮬레이션 — bkurs-be 예

```
Static asset:        50 GB × $0.025 = $1.25
CloudFront origin:   100 GB egress (CF) = 무료
User upload:         200 GB Intelligent-Tiering ~= $5
DB backup:           500 GB Glacier Deep ($0.002) = $1
CloudWatch log:      100 GB IA → Glacier Deep = $1.5
Audit:               50 GB Glacier Deep + Object Lock = $0.1
Build artifact:      30 GB Standard-IA = $0.4
Terraform state:     1 MB Standard = ~$0

Storage subtotal:    ~$9/월

GET (user image):    1M req × $0.00043 = $0.43
PUT (upload):        100K × $0.0054 = $0.54
GET via CF (no S3):  500M × $0 = 0
Cross-region DR repl: 100 GB × $0.09 = $9

Total:               ~$20/월
```

## 11. Cost Allocation Tag

```bash
aws s3api put-bucket-tagging --bucket bkurs-uploads \
  --tagging 'TagSet=[{Key=Environment,Value=prod},{Key=CostCenter,Value=eng},{Key=Service,Value=upload}]'

# 활성 (Cost Explorer)
aws ce update-cost-allocation-tags-status --cost-allocation-tags-status '[
  { "TagKey": "Environment", "Status": "Active" },
  { "TagKey": "CostCenter",  "Status": "Active" }
]'
```

→ Cost Explorer 에서 service / environment 별 visibility.

## 12. Storage Lens (cost / object analysis)

```bash
aws s3control put-storage-lens-configuration \
  --account-id $ACCT --config-id default \
  --storage-lens-configuration '{
    "Id": "default",
    "AccountLevel": {
      "ActivityMetrics": { "IsEnabled": true },
      "BucketLevel": {
        "ActivityMetrics": { "IsEnabled": true },
        "PrefixLevel": {
          "StorageMetrics": {
            "IsEnabled": true,
            "SelectionCriteria": { "MaxDepth": 5, "MinStorageBytesPercentage": 1.0 }
          }
        }
      }
    },
    "IsEnabled": true,
    "DataExport": {
      "S3BucketDestination": {
        "AccountId": "ACCT_ID",
        "Arn": "arn:aws:s3:::bkurs-lens-export",
        "Format": "CSV",
        "OutputSchemaVersion": "V_1"
      }
    }
  }'
```

→ daily metric (Free tier 28 metric / Advanced 35 metric paid).

## 13. CloudWatch Storage metric

```bash
aws cloudwatch get-metric-statistics --namespace AWS/S3 \
  --metric-name BucketSizeBytes \
  --dimensions Name=BucketName,Value=bkurs-uploads Name=StorageType,Value=StandardStorage \
  --start-time 2026-05-15T00:00:00Z --end-time 2026-05-22T00:00:00Z \
  --period 86400 --statistics Average
```

자동 metric:
- `BucketSizeBytes`
- `NumberOfObjects`

per-class breakdown:
- `StandardStorage`, `IntelligentTieringFA`, `GlacierStorage`, `DeepArchiveStorage` ...

## 14. 비용 절감 전략

```
1. Lifecycle → Standard → IA → Glacier 자동
2. Intelligent-Tiering 활용 (변동 패턴)
3. Multipart aborted lifecycle (7d expire)
4. CloudFront + S3 Origin (DT egress ↓)
5. VPC Endpoint (cross-region X)
6. Versioning 의 old version expire
7. Delete marker cleanup
8. Cold data → Glacier Deep ($0.002/GB)
9. Logging bucket 의 retention 정책
10. Storage Lens 로 hot/cold prefix 파악
11. S3 Express One Zone (high-throughput)
```

## 15. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Versioning 의 old version 삭제 안 함 | NoncurrentVersionExpiration |
| Delete marker 누적 | ExpiredObjectDeleteMarker |
| Aborted multipart 보이지 않지만 비용 | abort-incomplete-multipart |
| Standard 그대로 cold data | Lifecycle / IT |
| Cross-region DT cost 모름 | replication 필요한지 |
| Small file Intelligent-Tiering | 128 KB+ 만 |
| Request 비용 인지 못 함 (small file 대량) | batch / S3 Express |

## 16. 다음

- ▶ [[04-consistency-and-performance]]
- ▶ [[../02-access/01-bucket-policy-and-acl]]
- ▶ [[../03-lifecycle/01-lifecycle-policy-and-tiering]]

## 17. 영상 / 자료

| 자료 |
| --- |
| [S3 Pricing](https://aws.amazon.com/s3/pricing/) |
| [S3 Storage Lens](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage_lens.html) |

## 18. 참고

- [[../s3]]
- [[02-storage-classes]]
