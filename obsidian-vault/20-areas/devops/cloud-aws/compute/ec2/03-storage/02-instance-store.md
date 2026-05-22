---
title: "EC2 03-2 — Instance Store (NVMe local SSD)"
kind: knowledge
status: current
created_at: 2026-05-22T15:18:00+09:00
tags: [aws, ec2, storage, instance-store, nvme]
home_hub: ec2
---

# Instance Store (NVMe local SSD)

> Host 직접 NVMe. EBS 보다 10~50x 빠름, 그러나 **stop 시 데이터 손실**. cache / scratch / shuffle / temp 만.

## 1. EBS vs Instance Store

| | EBS gp3 | EBS io2 | Instance Store NVMe |
| --- | --- | --- | --- |
| 위치 | EBS Service (network) | EBS Service | host server 직접 |
| Latency | ~1ms | sub-ms | ~50μs |
| IOPS | 16K | 256K | 1M+ |
| Throughput | 1 GB/s | 4 GB/s | 25 GB/s+ |
| 영속 | ✓ | ✓ | **✗ stop / terminate 시 소실** |
| Snapshot | ✓ | ✓ | ✗ |
| Attach | 다른 instance OK | 다른 instance OK | host 의존 |
| 비용 | $0.10/GB·월 | $0.14/GB·월 + IOPS | 인스턴스 비용에 포함 |
| 한도 | volume size | size | instance type 별 고정 |

## 2. NVMe 가진 type

| Family | NVMe SSD |
| --- | --- |
| **i3 / i3en / i4i** | storage optimized (메인 용도) |
| **i4g / im4gn** | Graviton storage optimized |
| **d2 / d3 / d3en** | HDD dense (대용량) |
| **m5d / m6id / m7gd** | 범용 + NVMe |
| **c5d / c6id / c7gd** | 컴퓨팅 + NVMe |
| **r5d / r6id / r7gd** | 메모리 + NVMe |
| **z1d** | high frequency + NVMe |
| **x2idn / x2iedn** | 거대 RAM + NVMe |

예:
- i4i.large = 468 GB NVMe / 인스턴스 비용 $0.196/h
- i4i.32xlarge = 30 TB NVMe / $7.55/h
- d3.8xlarge = 192 TB HDD / $4.0/h
- m7gd.16xlarge = 3.8 TB / $3.4/h

## 3. 활성 / 확인

NVMe 는 instance 시작 시 자동 mount 안 됨. 확인:
```bash
lsblk
# nvme1n1  468G  (instance store)
# nvme0n1   30G  (EBS root)
```

확인 명령:
```bash
sudo nvme list
# instance store 는 model = "Amazon EC2 NVMe Instance Storage"
```

## 4. mount 설정

```bash
sudo mkfs -t xfs /dev/nvme1n1
sudo mkdir /mnt/local
sudo mount /dev/nvme1n1 /mnt/local
```

**`/etc/fstab` 에 절대 추가 X** — instance store device 이름이 host 마다 다를 수 있음. user-data 로 매번 자동 mount:

```bash
#!/bin/bash
set -e
DEV=$(lsblk -dno NAME,MODEL | awk '/Instance Storage/ {print "/dev/"$1}' | head -1)
if [ -n "$DEV" ]; then
  if ! blkid $DEV; then
    mkfs -t xfs -f $DEV
  fi
  mkdir -p /mnt/local
  mount $DEV /mnt/local
fi
```

## 5. 데이터 소실 시나리오

| 상황 | 데이터 |
| --- | --- |
| Reboot | ✓ 유지 |
| Hibernate | ✗ 소실 |
| Stop → Start | **✗ 소실** (다른 host 로 이동) |
| Terminate | ✗ 소실 |
| Underlying host failure | ✗ 소실 |
| AMI 생성 | ✗ 포함 안 됨 |

## 6. 적합한 워크로드

✓ **시나리오**
- Cache (Redis / Memcached — 메모리 보조)
- Shuffle / spill (Spark / Hadoop)
- Database temp space (PostgreSQL temp_tablespaces)
- Scratch space (compile / build)
- Bulk file processing (download → process → upload)

✗ **부적합**
- Primary DB
- 영구 사용자 데이터
- 단일 데이터 origin

## 7. RAID

여러 NVMe 가 있는 type 은 RAID 0 으로 묶기:
```bash
sudo mdadm --create --verbose /dev/md0 --level=0 \
  --raid-devices=4 /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1 /dev/nvme4n1
sudo mkfs -t xfs /dev/md0
sudo mount /dev/md0 /mnt/local
```

→ throughput 4x. data redundancy 없으니 RAID 0 만 (RAID 5/6 의미 없음 — host 가 실패하면 다 잃음).

## 8. TRIM / Discard

```bash
sudo mount -o discard /dev/nvme1n1 /mnt/local
# 또는 fstrim 주기
sudo fstrim /mnt/local
```

→ 사용 안 하는 block 해제 → write 성능 유지.

## 9. 성능 benchmark

```bash
fio --filename=/mnt/local/test --rw=randread --bs=4k --iodepth=256 \
    --runtime=60 --time_based --name=test --ioengine=libaio --direct=1 \
    --numjobs=8 --group_reporting

# i4i.large 예시: ~250K IOPS / ~1 GB/s
```

## 10. ASG + Instance Store 패턴

```
ASG launch:
  user-data → instance store 자동 format + mount /mnt/cache
  app 시작 → /mnt/cache 사용
ASG scale-in / failure:
  data 손실 → ASG 가 새 instance 시작 → 같은 user-data → 빈 cache
  → app 이 빈 cache 핸들링 가능해야 함
```

## 11. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| `/etc/fstab` 등록 → stop-start 시 부팅 실패 | user-data 로 매번 mount |
| Stop-Start = 데이터 손실 모름 | 영속 필요 시 EBS |
| RAID 5/6 적용 | RAID 0 만 (host fail 시 다 잃음) |
| AMI 생성하면 포함된다고 착각 | instance store 는 AMI 미포함 |
| 다른 instance 로 옮길 수 있다고 착각 | 옮기려면 S3/EBS 경유 |

## 12. 다음

- ▶ [[03-snapshots-ami-lifecycle]]
- ▶ [[../04-operations/01-ssm-session-manager]]
- ▶ [[../05-security/03-hardening]]

## 13. 영상 / 자료

| 자료 |
| --- |
| [Instance Store 공식](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/InstanceStorage.html) |
| [NVMe Performance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ssd-instance-store.html) |

## 14. 참고

- [[../ec2]]
- [[01-ebs-deep]]
- [[../01-fundamentals/02-instance-types]] §Storage Optimized
