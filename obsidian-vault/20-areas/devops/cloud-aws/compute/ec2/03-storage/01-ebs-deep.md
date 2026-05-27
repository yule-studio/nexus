---
title: "EC2 03-1 — EBS Deep (gp3 / io2 / st1 / sc1 + IOPS + Multi-Attach)"
kind: knowledge
status: current
created_at: 2026-05-22T15:16:00+09:00
tags: [aws, ec2, storage, ebs]
home_hub: ec2
---

# EBS Deep

> SAN-like block storage. Volume Type 6 + IOPS/Throughput 분리 + Multi-Attach + 암호화. AZ scope.

## 1. Volume Type

| Type | size | IOPS | Throughput | $/GB·월 | 권장 |
| --- | --- | --- | --- | --- | --- |
| **gp3** (SSD) | 1 GiB - 16 TiB | 3000 (free) up to 16K | 125 MB/s (free) up to 1000 | $0.096 | 일반 ★ |
| **gp2** (SSD) | 1 GiB - 16 TiB | 100 → 16K (size 따라) | size 따라 | $0.114 | 마이그 |
| **io2** (SSD) | 4 GiB - 64 TiB | up to 256K | 4000 MB/s | $0.142 + IOPS 가격 | DB 고성능 |
| **io2 Block Express** | up to 64 TiB | up to 256K | 4000 MB/s | premium | SAP HANA |
| **st1** (HDD) | 125 GiB - 16 TiB | low | up to 500 MB/s | $0.054 | big data sequential |
| **sc1** (HDD) | 125 GiB - 16 TiB | low | up to 250 MB/s | $0.018 | archive cold |

> gp2 EOL 권고 — **gp3 으로 마이그** 권장 (싸고 빠름).

## 2. gp3 — 권장 default

```
baseline:  3000 IOPS + 125 MB/s (size 무관, 무료)
추가 비용: $0.005/IOPS·월 (3000 초과) + $0.040/MB·s·월 (125 초과)
max:       16K IOPS / 1000 MB/s
```

| size | gp2 (free IOPS) | gp3 (free IOPS) |
| --- | --- | --- |
| 50 GiB | 150 (느림) | 3000 (빠름) |
| 100 GiB | 300 | 3000 |
| 1 TiB | 3000 | 3000 |
| 10 TiB | 16K | 3000 + provision |

→ 작은 volume 일수록 gp3 가 훨씬 우월.

## 3. io2 / io2 Block Express

```bash
aws ec2 create-volume --volume-type io2 --size 100 \
  --iops 10000 --availability-zone ap-northeast-2a \
  --multi-attach-enabled
```

| | io1 (EOL) | io2 | io2 Block Express |
| --- | --- | --- | --- |
| durability | 99.8-99.9% | 99.999% | 99.999% |
| IOPS:GB | 50 | 500 | 1000 |
| max IOPS | 64K | 256K | 256K |
| Multi-Attach | ✓ (Nitro) | ✓ | ✓ |
| sub-ms latency | ✗ | ✓ | ✓ |

→ DB primary / shared filesystem (GFS2 / Oracle RAC).

## 4. st1 / sc1 — HDD

| | st1 (Throughput Optimized HDD) | sc1 (Cold HDD) |
| --- | --- | --- |
| 용도 | log / big data sequential | archive |
| baseline | 40 MB/s/TiB | 12 MB/s/TiB |
| burst | up to 500 MB/s | up to 250 MB/s |
| 부팅 | ✗ | ✗ |

> random IO 안 됨. EMR / Kafka / 큰 log volume.

## 5. Volume 생성 / attach

```bash
aws ec2 create-volume --availability-zone ap-northeast-2a \
  --size 100 --volume-type gp3 \
  --iops 5000 --throughput 250 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=app-data}]'

aws ec2 attach-volume --volume-id vol-xxx --instance-id i-xxx --device /dev/sdf

# Instance 안
lsblk
sudo mkfs -t xfs /dev/nvme1n1
sudo mkdir /data && sudo mount /dev/nvme1n1 /data
echo '/dev/nvme1n1 /data xfs defaults,nofail 0 2' | sudo tee -a /etc/fstab
```

## 6. AZ scope — 다른 AZ 로 이동

```bash
# Snapshot 생성
aws ec2 create-snapshot --volume-id vol-xxx --description "for AZ move"

# 다른 AZ 에서 volume 복원
aws ec2 create-volume --snapshot-id snap-xxx \
  --availability-zone ap-northeast-2c --volume-type gp3
```

## 7. 암호화

```bash
# Default 활성
aws ec2 enable-ebs-encryption-by-default

# Specific KMS key
aws ec2 modify-ebs-default-kms-key-id --kms-key-id alias/ebs
```

- Volume / Snapshot 모두 암호화
- AMI 도 암호화 snapshot 으로 빌드
- 성능 영향 거의 없음 (Nitro offload)

## 8. Resize (확장)

```bash
aws ec2 modify-volume --volume-id vol-xxx --size 200 --iops 8000 --throughput 500

# Instance 안 (Linux)
sudo growpart /dev/nvme1n1 1     # partition resize
sudo xfs_growfs /data            # filesystem resize
```

| 한도 |
| --- |
| 6h 마다 한번 |
| 축소 불가 (snapshot → 작은 volume 복원) |

## 9. Multi-Attach

io1/io2 만. 같은 AZ 의 instance 최대 16:
```bash
aws ec2 create-volume --volume-type io2 --size 100 \
  --iops 10000 --multi-attach-enabled \
  --availability-zone ap-northeast-2a
```

> File system 이 cluster-aware 해야 안전 (GFS2 / OCFS2 / Oracle ASM). XFS / ext4 동시 mount = 데이터 corruption.

## 10. EBS-Optimized

7세대 = 기본. 하위 type 은 명시:
```bash
aws ec2 run-instances --ebs-optimized ...
```

→ EBS bandwidth 가 network 와 분리.

## 11. Performance Tuning

```bash
# Read-ahead
sudo blockdev --setra 4096 /dev/nvme1n1

# IO scheduler
echo "noop" | sudo tee /sys/block/nvme1n1/queue/scheduler  # NVMe 권장

# fio benchmark
fio --filename=/data/test --rw=randread --bs=4k --iodepth=64 \
    --runtime=60 --time_based --name=randread --ioengine=libaio --direct=1
```

## 12. Snapshot

다음 doc — [[03-snapshots-ami-lifecycle]]

## 13. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| gp2 그대로 사용 | gp3 마이그 (싸고 빠름) |
| 작은 volume IOPS 부족 | gp3 |
| io1 사용 (durability ↓) | io2 |
| AZ 잘못 → instance attach 불가 | 같은 AZ 또는 snapshot |
| Multi-Attach + ext4 = 데이터 corruption | cluster-aware FS 만 |
| 암호화 미활성 | default encryption |
| 6h resize 한도 모름 | 사전 계획 |

## 14. 비용 절감

- gp3 → gp2 대비 자체 -20%
- 사용 안 하는 volume / snapshot 정리 (DLM)
- AMI 정리 → snapshot 자동 삭제

## 15. CDK

```typescript
const vol = new ec2.Volume(this, 'Data', {
  availabilityZone: 'ap-northeast-2a',
  size: cdk.Size.gibibytes(100),
  volumeType: ec2.EbsDeviceVolumeType.GP3,
  iops: 5000, throughput: 250,
  encrypted: true,
});

new ec2.CfnVolumeAttachment(this, 'Att', {
  instanceId: instance.instanceId,
  volumeId: vol.volumeId,
  device: '/dev/sdf',
});
```

## 16. 다음

- ▶ [[02-instance-store]]
- ▶ [[03-snapshots-ami-lifecycle]]
- ▶ [[../../../storage/ebs/ebs]]

## 17. 영상 / 자료

| 자료 |
| --- |
| [EBS Deep Dive — re:Invent](https://www.youtube.com/watch?v=wsMWANWNoqQ) |
| [gp3 발표](https://aws.amazon.com/blogs/aws/new-amazon-ebs-gp3-volume-lets-you-provision-performance-apart-from-capacity/) |

## 18. 참고

- [[../ec2]]
- [[../01-fundamentals/02-instance-types]] §EBS-Optimized
