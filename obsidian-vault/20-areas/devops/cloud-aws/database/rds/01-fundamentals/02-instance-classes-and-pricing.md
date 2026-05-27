---
title: "RDS 01-2 — Instance Classes + Pricing (Graviton / Memory / Storage)"
kind: knowledge
status: current
created_at: 2026-05-22T19:54:00+09:00
tags: [aws, rds, fundamentals, pricing, instance, graviton, storage]
home_hub: rds
---

# RDS Instance Classes + Pricing

> db.t4g (burstable) / db.m6g (general) / db.r6g (memory) / db.x2g (huge RAM). Graviton ARM64 -20% 비용. gp3 default storage.

## 1. Instance Class Family

| Family | 용도 | 예 |
| --- | --- | --- |
| **db.t4g / db.t3** | burstable, dev/small prod | db.t4g.medium (2 vCPU, 4 GB) |
| **db.m6g / db.m6i** | balanced general | db.m6g.large (2 vCPU, 8 GB) |
| **db.r6g / db.r6i** | memory-optimized | db.r6g.large (2 vCPU, 16 GB) |
| **db.x2g / db.x2idn** | huge RAM | db.x2idn.16xlarge (64 vCPU, 1024 GB) |
| **db.c6g / db.c6i** | compute (RDS only) | db.c6g.xlarge (4 vCPU, 8 GB) |
| **db.z1d** | high frequency | db.z1d.large (2 vCPU, 16 GB, 4 GHz) |

## 2. Generation = 새로움

```
db.t3 (Intel) → db.t4g (Graviton 2)
db.m5 (Intel) → db.m6g (Graviton 2) / db.m6i (Intel Ice Lake)
db.r5 (Intel) → db.r6g (Graviton 2) / db.r6i (Intel)
```

> 같은 가격 = 최신 gen + Graviton 선택.

## 3. Graviton (ARM64)

| | x86 (db.r6i) | Graviton (db.r6g) |
| --- | --- | --- |
| 가격 | 100% | ~80% |
| 성능 | 100% | 110~135% |
| Engine 호환 | 다 | 다 (Postgres, MySQL, MariaDB) |
| SQL Server / Oracle | ✓ (x86 만) | ✗ |

> Postgres / MySQL = **Graviton 권장**. SQL Server / Oracle = x86.

## 4. Burstable t-class

```
db.t4g.medium:
  baseline 40% CPU credit + burst 100%
  CPU credit 부족 시:
    - Standard mode: baseline 강제
    - Unlimited mode: $0.05/vCPU·hour 추가 (default ON for RDS)
```

```bash
aws rds modify-db-instance --db-instance-identifier bkurs-prod \
  --cpu-credit-allocation-mode unlimited   # default
```

→ production 은 `unlimited` 권장.

## 5. Pricing — Seoul (ap-northeast-2)

| Instance | $/h | $/월 (24x7) |
| --- | --- | --- |
| db.t4g.micro | $0.018 | $13 |
| db.t4g.small | $0.036 | $26 |
| db.t4g.medium | $0.103 | $75 |
| db.t4g.large | $0.205 | $150 |
| db.m6g.large | $0.196 | $143 |
| db.m6g.xlarge | $0.392 | $286 |
| db.r6g.large | $0.252 | $184 |
| db.r6g.xlarge | $0.504 | $368 |
| db.r6g.16xlarge | $8.06 | $5880 |

| Multi-AZ | × 2 (standby) |
| Multi-AZ DB Cluster (Postgres / MySQL) | × 3 (1 writer + 2 readable standbys) |

## 6. Storage Pricing

| Type | $/GB·월 | IOPS | Throughput |
| --- | --- | --- | --- |
| gp3 ★ | $0.138 | 3000 free, +$0.024/IOPS | 125 MB/s free, +$0.084/MB·s |
| gp2 | $0.138 | size 따라 | size 따라 |
| io1 | $0.158 | $0.10/IOPS·월 | included |
| io2 Block Express | $0.158 + extra | $0.10/IOPS·월 | included |
| Magnetic (legacy) | $0.10 | low | low |

> gp3 권장. io2 = 256K IOPS 까지 필요한 경우.

## 7. Storage Auto Scaling

```bash
aws rds modify-db-instance --db-instance-identifier bkurs-prod \
  --max-allocated-storage 500
```

→ 80% 도달 시 자동 증가, max 500 GB 까지.

## 8. Backup Storage Pricing

```
Free: 같은 size 의 allocated storage 만큼
초과: $0.095/GB·월
Snapshot: $0.095/GB·월 (initial)
```

→ 100 GB DB + 35d 보존 = 보통 무료 안에서.

## 9. Reserved Instance

| 약정 | 할인 |
| --- | --- |
| 1y No Upfront | ~25% |
| 1y Partial Upfront | ~30% |
| 1y All Upfront | ~35% |
| 3y All Upfront | ~65% |

```bash
aws rds describe-reserved-db-instances-offerings \
  --db-instance-class db.t4g.medium \
  --duration 1y --product-description postgresql \
  --offering-type "All Upfront"

aws rds purchase-reserved-db-instances-offering \
  --reserved-db-instances-offering-id xxx \
  --db-instance-count 1
```

## 10. Aurora Serverless v2 비교

```
RDS db.t4g.medium 24x7:    $75/월
Aurora Serverless v2:
  0.5 ACU min - 2 ACU max
  0.5 ACU baseline → $0.12/h × 730 = $87/월
  
→ Idle 시 0.5 ACU = RDS 와 비슷. spike 흡수 = Serverless 유리.
```

## 11. 비용 절감 전략

```
1. Graviton (ARM64) → -20%
2. Right-sizing (Performance Insights 확인)
3. Reserved Instance 3y (-65%)
4. dev/staging 야간 stop (RDS 의 manual stop 7일까지)
5. gp3 → gp2 보다 싸고 빠름
6. Backup retention 35d 안 (free 안에서)
7. Read Replica 만 production 에 둠
8. Multi-AZ Cluster (3-node) 만 critical
9. Aurora Serverless v2 (variable workload)
10. Cross-region snapshot 만 DR
```

## 12. Hard limits

| | |
| --- | --- |
| DB instance / region / account | 40 (soft, 증액 가능) |
| Storage per instance | 64 TiB (gp3) |
| Max IOPS per volume | 256K (io2) |
| Read Replicas | 5 per primary |
| Multi-AZ DB Cluster | 2 readable standbys |
| Snapshot / region | 100 (soft) |

## 13. Storage Auto Scaling 의 한계

```
6h 마다 1회 (Cool Down)
26 GB or 10% (더 큰 값) 증가
max 64 TiB
```

## 14. CloudWatch Pricing metric

```bash
aws cloudwatch get-metric-statistics --namespace AWS/RDS \
  --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=bkurs-prod \
  --period 60 --statistics Average \
  --start-time $(date -u -v -1H +"%Y-%m-%dT%H:%M:%SZ") --end-time $(date -u +"%Y-%m-%dT%H:%M:%SZ")
```

| Metric | 의미 |
| --- | --- |
| `CPUUtilization` | CPU % |
| `DatabaseConnections` | connection 수 |
| `FreeableMemory` | 사용 가능 RAM |
| `FreeStorageSpace` | 남은 storage |
| `ReadIOPS` / `WriteIOPS` | IOPS |
| `ReadLatency` / `WriteLatency` | latency |
| `ReplicaLag` | Read Replica 지연 |
| `BurstBalance` | gp2 credit |
| `EBSByteBalance%` / `EBSIOBalance%` | network burst |

## 15. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Burstable production 의 credit 고갈 | unlimited 또는 m-class |
| gp2 (size 따라 IOPS) → 작은 size 느림 | gp3 |
| Storage Auto Scaling 6h cool down | 미리 증액 |
| db.t4g 의 ARM 호환 | Postgres / MySQL OK, SQL Server X |
| Reserved 3y 약정 후 size 변경 | family 동일 안 함 |
| Stop 7일 후 자동 start (RDS) | dev/staging 만 |

## 16. 다음

- ▶ [[03-multi-az-and-read-replica]]
- ▶ [[../02-operations/01-backup-and-pitr]]
- ▶ [[../02-operations/02-performance-and-monitoring]]

## 17. 영상 / 자료

| 자료 |
| --- |
| [RDS Pricing](https://aws.amazon.com/rds/pricing/) |
| [Instance Classes](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.DBInstanceClass.html) |
| [Graviton — RDS](https://aws.amazon.com/blogs/aws/new-graviton2-and-graviton3-based-database-instances/) |

## 18. 참고

- [[../rds]]
- [[01-engine-and-versions]]
