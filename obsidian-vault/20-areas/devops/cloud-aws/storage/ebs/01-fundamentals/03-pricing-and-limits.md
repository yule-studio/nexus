---
title: "EBS 01-3 — Pricing + Limits"
kind: knowledge
status: current
created_at: 2026-05-22T19:19:00+09:00
tags: [aws, ebs, fundamentals, pricing, limits]
home_hub: ebs
---

# EBS Pricing + Limits

> Storage + IOPS + Throughput + Snapshot + Cross-region DT. Hard limits 표.

## 1. 가격 구성

```
Volume storage (GB·월)
+ Provisioned IOPS (io1/io2, gp3 의 3000+)
+ Provisioned Throughput (gp3 의 125+ MB/s)
+ Snapshot storage (changed block, S3)
+ Snapshot DT cross-region
+ FSR (활성 만큼)
```

## 2. Volume Pricing — Seoul (ap-northeast-2)

| Type | $/GB·월 | IOPS | Throughput |
| --- | --- | --- | --- |
| gp3 | $0.096 | 3000 free, +$0.005/IOPS | 125 MB/s free, +$0.04/MB·s |
| gp2 | $0.114 | size 따라 | size 따라 |
| io2 | $0.142 | $0.065/IOPS·월 | included |
| io2 Block Express | premium | included | included |
| st1 | $0.054 | (HDD) | (HDD) |
| sc1 | $0.018 | (HDD cold) | (HDD cold) |

## 3. gp3 비용 시뮬레이션

```
gp3 100 GB:
  100 × $0.096 = $9.60/월

gp3 100 GB + 5000 IOPS + 250 MB/s:
  100 × $0.096 = $9.60
  (5000-3000) × $0.005 = $10
  (250-125) × $0.04 = $5
  = $24.60/월
```

vs gp2 100 GB:
```
100 × $0.114 = $11.40/월
+ 300 IOPS baseline (느림)
```

> gp3 가 같은 가격에 16x IOPS + 8x throughput. **무조건 gp3**.

## 4. io2 비용 시뮬레이션

```
io2 500 GB + 50000 IOPS:
  500 × $0.142 = $71
  50000 × $0.065 = $3250
  = $3321/월
```

→ io2 = 고성능 DB 만. gp3 로 대체 가능하면 비용 100x 차이.

## 5. Snapshot Pricing

| | |
| --- | --- |
| Standard tier | $0.05/GB·월 (changed block) |
| Archive tier | $0.0125/GB·월 (75% 절감) |
| Archive activation | $0.0025/GB |
| Cross-region copy | DT $0.09/GB (one-time) + dest storage |
| FSR | $0.75/h/snapshot/AZ |

### Snapshot 비용 효율
- Snapshot = changed block 만 (first 만 full)
- 동일 volume 의 10 daily snapshot = 첫 + 매일 변경분
- 100 GB volume × 10 snapshot ≠ 1000 GB (보통 ~150 GB)

## 6. Hard Limits

| 항목 | Limit |
| --- | --- |
| Volume size | 1 GiB - 16 TiB (gp3/gp2) |
| io2/Block Express size | 4 GiB - 64 TiB |
| st1/sc1 size | 125 GiB - 16 TiB |
| Max IOPS | 256K (io2 Block Express) |
| Max Throughput | 4000 MB/s |
| Volumes / region | 5000 (soft) |
| Total provisioned storage / region | 50 TiB (soft) |
| Snapshots / region | 100K |
| Volume attach / instance | type 따라 (Nitro 28+) |
| Multi-Attach instances | 16 |
| Modify volume | 6h 마다 1번 |
| Snapshot copy / region | 5 concurrent (soft) |
| FSR | 5/region (soft) |

## 7. Service Quotas — 증액

```bash
aws service-quotas list-service-quotas --service-code ebs \
  --query 'Quotas[?Adjustable==`true`].[QuotaName,Value,QuotaCode]' --output table

# 증액 요청
aws service-quotas request-service-quota-increase \
  --service-code ebs \
  --quota-code L-D18FCD1D \
  --desired-value 200
```

## 8. Cost Allocation Tag

```bash
aws ec2 create-tags --resources vol-xxx snap-xxx \
  --tags Key=Environment,Value=prod \
         Key=Service,Value=postgresql \
         Key=CostCenter,Value=engineering

# 활성 (Cost Explorer)
aws ce update-cost-allocation-tags-status --cost-allocation-tags-status '[
  { "TagKey": "Environment", "Status": "Active" }
]'
```

## 9. CloudWatch metric

```
VolumeReadOps / VolumeWriteOps          (IOPS)
VolumeReadBytes / VolumeWriteBytes      (Throughput)
VolumeQueueLength                       (대기열)
VolumeIdleTime
VolumeTotalReadTime / VolumeTotalWriteTime
BurstBalance                            (gp2 / st1 / sc1)
```

```bash
aws cloudwatch get-metric-statistics --namespace AWS/EBS \
  --metric-name VolumeReadOps \
  --dimensions Name=VolumeId,Value=vol-xxx \
  --start-time 2026-05-15T00:00:00Z \
  --end-time   2026-05-22T00:00:00Z \
  --period 3600 --statistics Sum,Average
```

## 10. 비용 절감 패턴

```
1. gp2 → gp3 marination (자체 20% 절감)
2. Right-sizing (under-utilized volume detect)
3. Snapshot lifecycle (DLM + retention 14d)
4. Old snapshot Archive Tier
5. DR snapshot 만 cross-region (not all)
6. FSR 만 critical (always-on snapshot)
7. Unused volume 정리 (Cost Optimizer)
8. DeleteOnTermination 활성 (orphan volume 방지)
9. EBS direct API 로 backup tool 직접 효율화
10. io1 → io2 (durability ↑ + 가격 동일)
```

## 11. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| gp2 계속 사용 | gp3 |
| Detached volume 비용 인지 못 함 | EBS Cost Optimizer |
| Snapshot lifecycle 없음 | DLM |
| Cross-region copy 비용 폭주 | 필요한 것만 |
| FSR 모든 snapshot | critical 만 |
| io2 의 per-IOPS 비용 모름 | gp3 로 대체 가능한지 확인 |
| Archive Tier 90일 미만 삭제 | 만료 후 |

## 12. AWS Compute Optimizer

```bash
aws compute-optimizer get-ebs-volume-recommendations \
  --volume-arns arn:aws:ec2:ap-northeast-2:ACCT:volume/vol-xxx
```

→ 자동 right-sizing 추천 (provisioned IOPS / throughput 줄임).

## 13. 다음

- ▶ [[../02-operations/01-encryption-kms]]
- ▶ [[../02-operations/02-multi-attach-and-fast-snapshot-restore]]
- ▶ [[../02-operations/03-performance-and-monitoring]]

## 14. 영상 / 자료

| 자료 |
| --- |
| [EBS Pricing](https://aws.amazon.com/ebs/pricing/) |
| [EBS Service Quotas](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-resource-quotas.html) |
| [Compute Optimizer EBS](https://docs.aws.amazon.com/compute-optimizer/latest/ug/view-ebs-recommendations.html) |

## 15. 참고

- [[../ebs]]
- [[01-volume-types-deep]]
- [[02-snapshot-and-ami]]
