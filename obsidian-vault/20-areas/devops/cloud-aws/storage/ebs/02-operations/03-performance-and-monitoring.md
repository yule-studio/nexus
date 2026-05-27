---
title: "EBS 02-3 — Performance + Monitoring + Tuning"
kind: knowledge
status: current
created_at: 2026-05-22T19:25:00+09:00
tags: [aws, ebs, operations, performance, monitoring, fio, tuning]
home_hub: ebs
---

# EBS Performance + Monitoring

> IOPS + Throughput + Latency + Queue depth. Burst Balance (gp2/st1/sc1) + fio benchmark + CloudWatch metric + tuning.

## 1. EBS Performance 의 3 차원

```
IOPS:        per second 의 operation 수 (small random IO 중요)
Throughput:  bytes per second (large sequential IO 중요)
Latency:     operation 의 응답 시간 (sub-ms ~ ms)
```

| 워크로드 | 강조 |
| --- | --- |
| Database (random) | IOPS + latency |
| Log / Streaming | Throughput |
| Analytics scan | Throughput |
| Boot / OS | balanced |

## 2. IOPS / Throughput 의 관계

```
IOPS × IO size ≈ Throughput

예: 16000 IOPS × 16 KB = 256 MB/s
```

→ 작은 IO 가 많으면 IOPS bottleneck. 큰 IO 가 적으면 throughput bottleneck.

## 3. Burst Balance (gp2 / st1 / sc1)

```
gp2: 3 IOPS/GB baseline, max 3000 burst, credit (5.4M)
st1: 40 MB/s/TiB baseline, max 500 burst, credit (1 TB)
sc1: 12 MB/s/TiB baseline, max 250 burst, credit (160 GB)
```

CloudWatch metric `BurstBalance` < 50% 이면 burst credit 부족 위험.

> gp3 = burst 없음 (3000 IOPS 항상 보장).

## 4. EBS-Optimized

7세대 이상 Nitro = 자동. EBS bandwidth network 와 분리.

```
m7g.large:        EBS 최대 10 Gbps / 펌 1 Gbps
m7g.16xlarge:     EBS 50 Gbps
r6id.32xlarge:    EBS 60 Gbps
hpc7g.16xlarge:   EBS 25 Gbps
```

→ instance bandwidth 가 volume IOPS / Throughput 의 hard ceiling.

## 5. fio benchmark

```bash
# Sequential write
fio --filename=/data/test --rw=write --bs=1M --size=10G \
    --numjobs=1 --runtime=60 --time_based --name=seq-write \
    --ioengine=libaio --direct=1

# Sequential read
fio --filename=/data/test --rw=read --bs=1M --size=10G \
    --numjobs=1 --runtime=60 --time_based --name=seq-read \
    --ioengine=libaio --direct=1

# Random write (DB workload)
fio --filename=/data/test --rw=randwrite --bs=4k --size=10G \
    --numjobs=8 --iodepth=64 --runtime=60 --time_based \
    --name=rand-write --ioengine=libaio --direct=1 \
    --group_reporting

# Random read
fio --filename=/data/test --rw=randread --bs=4k --size=10G \
    --numjobs=8 --iodepth=64 --runtime=60 --time_based \
    --name=rand-read --ioengine=libaio --direct=1 \
    --group_reporting
```

## 6. 결과 해석

```
read: IOPS=10.5k, BW=42.0MiB/s
        ↑              ↑
       4K IO 의 IOPS   bandwidth
       
clat percentiles (usec):
  99.50th: 4.5ms
  99.90th: 12ms
  99.99th: 50ms
```

> P99 latency = production 의 user experience.

## 7. CloudWatch metric

| Metric | 의미 |
| --- | --- |
| `VolumeReadOps` / `VolumeWriteOps` | IOPS |
| `VolumeReadBytes` / `VolumeWriteBytes` | Throughput |
| `VolumeIdleTime` | idle 시간 |
| `VolumeQueueLength` | 대기열 (sat 시 IOPS 부족) |
| `VolumeTotalReadTime` / WriteTime | 총 시간 |
| `BurstBalance` | gp2/st1/sc1 credit |
| `VolumeStalledIOCheckFailed` | IO stalled |
| `VolumeConsumedReadWriteOps` | gp3/io2 의 IOPS 사용량 (provisioned 대비) |

### Latency 계산
```
Average Read Latency = VolumeTotalReadTime / VolumeReadOps
```

## 8. Alarm 표준

```bash
# Queue length 높음 (IOPS bottleneck)
aws cloudwatch put-metric-alarm --alarm-name ebs-queue \
  --metric-name VolumeQueueLength --namespace AWS/EBS \
  --statistic Average --period 60 --evaluation-periods 5 \
  --threshold 4 --comparison-operator GreaterThanThreshold \
  --dimensions Name=VolumeId,Value=vol-xxx

# Burst balance (gp2)
aws cloudwatch put-metric-alarm --alarm-name ebs-burst-low \
  --metric-name BurstBalance --namespace AWS/EBS \
  --statistic Minimum --period 300 --evaluation-periods 3 \
  --threshold 50 --comparison-operator LessThanThreshold \
  --dimensions Name=VolumeId,Value=vol-xxx
```

## 9. Tuning — OS

### IO scheduler
```bash
# NVMe = none (multi-queue 자동)
echo none | sudo tee /sys/block/nvme1n1/queue/scheduler

# Confirm
cat /sys/block/nvme1n1/queue/scheduler
```

### Read-ahead
```bash
sudo blockdev --setra 4096 /dev/nvme1n1
```

### Filesystem mount options
```bash
# XFS
sudo mount -o noatime,nodiratime,inode64 /dev/nvme1n1 /data

# ext4
sudo mount -o noatime,nodiratime,data=writeback /dev/nvme1n1 /data
```

### LVM stripe (multiple volumes)
```bash
sudo pvcreate /dev/nvme1n1 /dev/nvme2n1
sudo vgcreate data /dev/nvme1n1 /dev/nvme2n1
sudo lvcreate -i 2 -I 256K -l 100%FREE -n stripe data
sudo mkfs.xfs /dev/data/stripe
```

→ 2 volume = 2x IOPS / Throughput.

## 10. RAID 0 (multiple volumes)

```bash
sudo mdadm --create /dev/md0 --level=0 --raid-devices=2 \
  /dev/nvme1n1 /dev/nvme2n1
sudo mkfs -t xfs /dev/md0
```

→ throughput 2배. Redundancy 없음 (EBS 가 이미 11×9 durability 하므로 OK).

## 11. EBS Volume Pre-warming (옛 권고)

```
이전 (Xen-based):
  - 처음 read 시 lazy block init → 느림
  - 해결: dd if=/dev/zero of=/dev/nvme1n1
  
Nitro 이후:
  - 자동 — pre-warming 불필요
```

## 12. Performance 모니터링 도구

| 도구 | 용도 |
| --- | --- |
| `iostat -xz 1` | per-device 분석 |
| `iotop` | per-process IO |
| `dstat -d --top-io` | per-process top IO |
| `fio` | benchmark |
| `pidstat -d` | process 별 |
| AWS CloudWatch | metric |
| AWS Compute Optimizer | right-sizing 추천 |

```bash
iostat -xz 1
# Device: rrqm/s wrqm/s r/s w/s rkB/s wkB/s avgrq-sz avgqu-sz await r_await w_await %util
# nvme1n1:    0    0   100 50  4000  2000  80      0.8    8     6      10     85
                                                   ↑
                                              %util 85% = 거의 saturated
```

## 13. IOPS 한도 시뮬레이션

```bash
# gp3 100 GB, 3000 IOPS 무료
# 부하: random read 4K, 4 thread, iodepth 64

fio --filename=/data/test --rw=randread --bs=4k \
    --numjobs=4 --iodepth=64 --runtime=60 --time_based \
    --name=test --ioengine=libaio --direct=1 \
    --group_reporting

# 결과
# read: IOPS=3000, BW=12MiB/s (한도 도달)

# IOPS 늘림 (8000)
aws ec2 modify-volume --volume-id vol-xxx --iops 8000

# 다시 benchmark
fio ... 
# read: IOPS=8000
```

## 14. Connection / Instance bandwidth

```
m7g.large:        EBS 10 Gbps / 펌 1 Gbps
                  → max ~131 K IOPS (16K IO) or 1 GB/s

m7g.16xlarge:     EBS 50 Gbps / 펌 6.25 Gbps
                  → max ~256 K IOPS or 6 GB/s
```

→ Volume IOPS 가 instance bandwidth 보다 크면 의미 없음. Instance 도 right-sizing.

## 15. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| gp2 burst 고갈 → 느림 | gp3 |
| Instance bandwidth ↑ 안 함 | larger instance |
| ext4 default mount | noatime + writeback |
| IO scheduler `mq-deadline` | NVMe 는 none |
| 단일 volume IOPS 한도 | LVM stripe / RAID 0 |
| Queue length 무시 | alarm |
| pre-warming (불필요) | Nitro 는 자동 |

## 16. 다음

- ▶ [[../practice/01-volume-resize-snapshot]]
- ▶ [[../practice/02-encrypted-multi-attach]]
- ▶ [[../../efs/efs]]

## 17. 영상 / 자료

| 자료 |
| --- |
| [EBS Performance 공식](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSPerformance.html) |
| [fio](https://fio.readthedocs.io) |
| [Compute Optimizer EBS](https://docs.aws.amazon.com/compute-optimizer/latest/ug/view-ebs-recommendations.html) |

## 18. 참고

- [[../ebs]]
- [[../01-fundamentals/01-volume-types-deep]]
- [[01-encryption-kms]]
