---
title: "EFS 01-3 — Pricing + Performance"
kind: knowledge
status: current
created_at: 2026-05-22T19:38:00+09:00
tags: [aws, efs, fundamentals, pricing, performance]
home_hub: efs
---

# EFS Pricing + Performance

> Storage + Throughput (Elastic / Provisioned) + Replication. Latency 1-2 ms (GP) + Throughput 1 GB/s+ (Elastic).

## 1. Pricing — Seoul (ap-northeast-2)

| 항목 | $ |
| --- | --- |
| Standard storage | $0.30/GB·월 |
| One Zone storage | $0.20/GB·월 |
| Standard-IA storage | $0.0125/GB·월 |
| One Zone-IA storage | $0.008/GB·월 |
| Standard-Archive | $0.005/GB·월 |
| One Zone-Archive | $0.003/GB·월 |
| IA Retrieval | $0.01/GB |
| Archive Retrieval | $0.03/GB |
| Provisioned Throughput | $6.00/MB·s·월 |
| Elastic Throughput Read | $0.03/GB |
| Elastic Throughput Write | $0.06/GB |
| Replication | DT (source) + dest storage |
| Backup (AWS Backup) | $0.05/GB·월 |

## 2. 비용 시뮬레이션 — 1 TB shared

```
Standard 전체:
  1024 × $0.30 = $307/월
+ Elastic Throughput (50 GB read/일):
  50 × 30 × $0.03 = $45/월
= $352/월

Lifecycle 적용 (Hot 10% / IA 40% / Archive 50%):
  102 × $0.30 = $31
  410 × $0.0125 = $5
  512 × $0.005 = $2.5
+ Throughput = $45
= $83/월 (-76%)
```

> Lifecycle = 가장 큰 절감.

## 3. Performance 차원

```
Latency:    1-2 ms (General Purpose, same-AZ)
            4-10 ms (cross-AZ)
            10ms+ (MaxIO)

IOPS:       35K ops/s (GP Per-FS) 또는 unlimited (MaxIO)
Throughput: 1 GB/s+ (Elastic) 또는 fixed (Provisioned)
```

## 4. Throughput Burst Credit (Bursting mode)

```
Baseline: 50 KB/s/GB
+ Burst (credit 기반):
  매일 2.1 MB/s/GB 까지 burst 가능
  credit 부족 시 baseline 강제
```

```
1 GB:   50 KB/s baseline
100 GB: 5 MB/s baseline + 211 MB/s burst
1 TB:   50 MB/s baseline + 2.1 GB/s burst (한도)
```

> **작은 filesystem (< 1 TB) = throughput 항상 부족**. Elastic 으로.

## 5. Elastic 의 동작

```
Auto-scale 1 GB/s → 10+ GB/s
사용한 만큼만 과금
no provisioning
spike 대응 자동
```

활용: 변동 큰 workload, sparse access pattern.

## 6. Provisioned Throughput

```bash
aws efs update-file-system --file-system-id $FS_ID \
  --throughput-mode provisioned \
  --provisioned-throughput-in-mibps 1000
```

```
$6/MB·s·월
1000 MB/s = $6000/월
```

활용: 일정 baseline 큰 throughput + cost predictability.

## 7. CloudWatch metric

| Metric | 의미 |
| --- | --- |
| `StorageBytes` (Standard / IA / Archive) | storage by class |
| `ClientConnections` | concurrent NFS client |
| `PermittedThroughput` | 현재 사용 가능한 throughput |
| `MeteredIOBytes` | Elastic 의 read/write |
| `TotalIOBytes` | 모든 IO |
| `BurstCreditBalance` | Bursting credit |
| `PercentIOLimit` | GP 의 35K 도달 % |
| `ProvisionedThroughputInMibps` | provisioned 값 |

## 8. Alarm 표준

```bash
# Burst credit 부족
aws cloudwatch put-metric-alarm --alarm-name efs-burst-low \
  --metric-name BurstCreditBalance --namespace AWS/EFS \
  --statistic Minimum --period 300 --evaluation-periods 3 \
  --threshold 1099511627776 --comparison-operator LessThanThreshold \
  --dimensions Name=FileSystemId,Value=fs-xxx

# GP IO limit
aws cloudwatch put-metric-alarm --alarm-name efs-io-limit \
  --metric-name PercentIOLimit --namespace AWS/EFS \
  --statistic Average --period 60 --evaluation-periods 5 \
  --threshold 80 --comparison-operator GreaterThanThreshold

# Storage 폭주
aws cloudwatch put-metric-alarm --alarm-name efs-storage \
  --metric-name StorageBytes --namespace AWS/EFS \
  --statistic Average --period 86400 --evaluation-periods 1 \
  --threshold 1099511627776 --comparison-operator GreaterThanThreshold \
  --dimensions Name=FileSystemId,Value=fs-xxx Name=StorageClass,Value=Standard
```

## 9. Mount option 최적화

```bash
# 기본
sudo mount -t efs -o tls $FS_ID:/ /mnt/efs

# 큰 throughput
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 \
  $FS_ID.efs.region.amazonaws.com:/ /mnt/efs

# Read-only
sudo mount -t efs -o tls,ro $FS_ID:/ /mnt/efs
```

| 옵션 | 의미 |
| --- | --- |
| `nfsvers=4.1` | NFS v4.1 (권장) |
| `rsize=1048576` | read buffer 1 MB |
| `wsize=1048576` | write buffer 1 MB |
| `hard` | server 응답 대기 (data integrity) |
| `timeo=600` | timeout 60s |
| `retrans=2` | retry 2 |
| `noresvport` | non-privileged port |
| `tls` | TLS encryption in transit |

## 10. fio benchmark

```bash
# Sequential write (multi-client)
fio --filename=/mnt/efs/test --rw=write --bs=1M --size=10G \
    --numjobs=4 --iodepth=8 --runtime=60 --time_based \
    --name=seq --ioengine=libaio --direct=1 --group_reporting

# Random read
fio --filename=/mnt/efs/test --rw=randread --bs=4k --size=10G \
    --numjobs=16 --iodepth=64 --runtime=60 --time_based \
    --name=rand --ioengine=libaio --direct=1 --group_reporting
```

전형적:
```
Elastic mode + 16 client:
  IOPS ~30K, BW ~600 MB/s

GP + Bursting (small FS):
  Burst credit 있을 때: 100 MB/s
  Burst credit 부족: 5 MB/s
```

## 11. Hard limits

| | |
| --- | --- |
| Filesystems / region / account | 1000 (soft) |
| File size | 47.9 TiB |
| Concurrent NFS clients | 25K-50K per Mount Target |
| Throughput | 10+ GiB/s (Elastic), 5 GiB/s (Provisioned) |
| Mount Targets / FS | 1 per AZ |
| Access Points / FS | 1000 |
| Tags / FS | 50 |

## 12. EFS vs FSx 비교

| | EFS | FSx for Lustre | FSx for Windows | FSx for NetApp ONTAP |
| --- | --- | --- | --- | --- |
| Protocol | NFS v4 | Lustre (HPC) | SMB / Windows | NFS + SMB |
| Performance | 1 GB/s+ | 100 GB/s+ | 1 GB/s+ | 2 GB/s+ |
| Latency | ms | μs | ms | sub-ms |
| Cost | $0.30/GB | $0.14+/GB | $0.13+/GB | $0.30+/GB |
| 용도 | shared file | HPC / ML | Active Directory / SMB | enterprise NAS |

## 13. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Bursting 작은 FS → 느림 | Elastic |
| Provisioned 과대 → 비용 | Elastic |
| TLS 없음 (in-transit 평문) | mount option `tls` |
| Mount Target 1 AZ 만 → cross-AZ DT | 모든 AZ |
| Storage class 분포 모름 | CloudWatch storage by class |
| Concurrent client 한도 | 큰 cluster = Provisioned 검토 |

## 14. 다음

- ▶ [[../02-operations/01-access-points-and-iam]]
- ▶ [[../02-operations/02-backup-and-replication]]
- ▶ [[../02-operations/03-mount-and-csi]]

## 15. 영상 / 자료

| 자료 |
| --- |
| [EFS Pricing](https://aws.amazon.com/efs/pricing/) |
| [EFS Performance](https://docs.aws.amazon.com/efs/latest/ug/performance.html) |

## 16. 참고

- [[../efs]]
- [[01-architecture-and-modes]]
- [[02-storage-classes-and-lifecycle]]
