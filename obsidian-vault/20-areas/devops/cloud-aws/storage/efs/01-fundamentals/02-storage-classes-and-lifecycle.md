---
title: "EFS 01-2 — Storage Classes + Lifecycle (Standard / IA / Archive)"
kind: knowledge
status: current
created_at: 2026-05-22T19:34:00+09:00
tags: [aws, efs, fundamentals, storage-class, lifecycle, ia, archive]
home_hub: efs
---

# EFS Storage Classes + Lifecycle

> 3 class — Standard / IA (Infrequent Access) / Archive (2024+). Lifecycle 자동 이동 + Retrieval cost.

## 1. 3 class 비교

| Class | Durability | AZ | Retrieval Latency | $/GB·월 | Retrieval $/GB |
| --- | --- | --- | --- | --- | --- |
| **Standard** | 11×9 | 3+ | ms | $0.30 | $0 |
| **One Zone Standard** | 11×9 | 1 | ms | $0.20 | $0 |
| **Infrequent Access (IA)** | 11×9 | 3+ | ms | $0.0125 (Standard-IA) / $0.008 (One Zone-IA) | $0.01/GB |
| **Archive (2024+)** | 11×9 | 3+ | min-h | $0.005 (Standard-Archive) / $0.003 (One Zone-Archive) | $0.03/GB |

> Standard 의 60x 저렴한 Archive.

## 2. Lifecycle Policy

```bash
aws efs put-lifecycle-configuration --file-system-id fs-xxx \
  --lifecycle-policies '[
    { "TransitionToIA": "AFTER_30_DAYS" },
    { "TransitionToArchive": "AFTER_90_DAYS" },
    { "TransitionToPrimaryStorageClass": "AFTER_1_ACCESS" }
  ]'
```

| 옵션 | 의미 |
| --- | --- |
| `TransitionToIA` | 1/7/14/30/60/90 days |
| `TransitionToArchive` | 90/180/365 days (Standard-Archive 만) |
| `TransitionToPrimaryStorageClass` | AFTER_1_ACCESS (access 시 자동 Standard 로) |

## 3. 동작

```
File 생성 → Standard 저장
  ↓ 30일 access 없음
  → IA 로 자동 이동
  ↓ access (read/write)
  → Standard 로 복귀 (Retrieval cost $0.01/GB)
  ↓ 다시 30일 access 없음
  → IA
  ↓ 90일 access 없음
  → Archive
  ↓ access (read/write)
  → Standard 로 복귀 (Retrieval cost $0.03/GB)
```

## 4. 비용 절감 시뮬레이션

```
1 TB filesystem
- Frequent access: 100 GB Standard (10%)
- Infrequent:      400 GB IA      (40%)
- Archive:         500 GB Archive (50%)

비용:
- 100 GB × $0.30 = $30
- 400 GB × $0.0125 = $5
- 500 GB × $0.005 = $2.5
= $37.5/월

vs 모두 Standard:
- 1024 × $0.30 = $307/월

→ 87% 절감
```

## 5. Retrieval cost

| | |
| --- | --- |
| IA → access | $0.01/GB |
| Archive → access | $0.03/GB |
| Same-AZ access | 무료 (One Zone-IA / Archive) |

> **자주 access 하면 retrieval cost 가 storage 절감 보다 클 수 있음**. access pattern 측정 필수.

## 6. CloudWatch metric

```bash
aws cloudwatch get-metric-statistics --namespace AWS/EFS \
  --metric-name StorageBytes \
  --dimensions Name=FileSystemId,Value=$FS_ID Name=StorageClass,Value=Standard \
  --start-time 2026-05-15T00:00:00Z --end-time 2026-05-22T00:00:00Z \
  --period 86400 --statistics Average

# IA / Archive 도
--dimensions Name=FileSystemId,Value=$FS_ID Name=StorageClass,Value=IA
--dimensions Name=FileSystemId,Value=$FS_ID Name=StorageClass,Value=Archive
```

## 7. Lifecycle 실제 적용 시점

```
EFS lifecycle:
- 정해진 시간에 schedule (UTC 09:00)
- "AFTER_30_DAYS" = 30일 idle
- 한번에 다 transition 안 됨 — gradual
- 매일 evaluate
```

## 8. Access Pattern 분석 — CloudWatch

```
metric: PercentageOfFilesAccessed
- 5% 이하 = IA 자체 적합
- 30%+ = retrieval cost 많음, Standard 유지

metric: PermittedThroughput
- 매일 변동 보고 mode 결정
```

## 9. lifecycle 전략 예 — bkurs-be

```
WordPress / CMS:
- TransitionToIA: 30 days
- TransitionToPrimaryStorageClass: AFTER_1_ACCESS

CI build artifact:
- TransitionToIA: 14 days
- TransitionToArchive: 90 days
- Standard 복귀: AFTER_1_ACCESS

ML model archive:
- TransitionToIA: 60 days  (자주 사용)
- TransitionToArchive: 180 days

Compliance log:
- TransitionToIA: 1 day
- TransitionToArchive: 30 days
- Object Lock 으로 영구
```

## 10. EFS Replication 의 lifecycle

```
Source: Standard + IA + Archive
Destination: replication 시 같은 lifecycle 적용 OK
            (Region replication 마다 별도)
```

## 11. Lifecycle policy 변경

```bash
# 새 policy
aws efs put-lifecycle-configuration --file-system-id fs-xxx \
  --lifecycle-policies '[
    { "TransitionToIA": "AFTER_60_DAYS" }
  ]'

# Disable lifecycle
aws efs put-lifecycle-configuration --file-system-id fs-xxx \
  --lifecycle-policies '[]'
```

> Existing IA / Archive object 는 lifecycle 변경 시 자동 이동 안 됨. Access 후 자연스러운 이동.

## 12. EFS Replication + Lifecycle

```bash
aws efs create-replication-configuration --source-file-system-id fs-src \
  --destinations Region=ap-northeast-1
```

destination 의 filesystem 도 별도 lifecycle 설정 가능.

자세히 → [[../02-operations/02-backup-and-replication]]

## 13. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Lifecycle 없음 → storage 폭주 | 항상 설정 |
| Retrieval cost 모름 → IA 자주 access | access pattern 측정 |
| Archive 의 retrieval latency 모름 | min-h, 즉시 X |
| `TransitionToPrimaryStorageClass` 없음 → IA 계속 retrieval cost | AFTER_1_ACCESS |
| Lifecycle 변경 시 기존 안 따라 옴 | access 또는 수동 |
| Replication + lifecycle 별도 | 각 region 마다 |

## 14. 다음

- ▶ [[03-pricing-and-performance]]
- ▶ [[../02-operations/01-access-points-and-iam]]
- ▶ [[../02-operations/02-backup-and-replication]]

## 15. 영상 / 자료

| 자료 |
| --- |
| [EFS Lifecycle](https://docs.aws.amazon.com/efs/latest/ug/lifecycle-management-efs.html) |
| [EFS Archive 발표](https://aws.amazon.com/blogs/aws/new-amazon-efs-archive-storage-class/) |

## 16. 참고

- [[../efs]]
- [[01-architecture-and-modes]]
