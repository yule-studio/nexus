---
title: "EBS 01-1 — Volume Types Deep (gp3 / io2 / st1 / sc1 / Block Express)"
kind: knowledge
status: current
created_at: 2026-05-22T19:15:00+09:00
tags: [aws, ebs, fundamentals, gp3, io2, block-express]
home_hub: ebs
---

# EBS Volume Types Deep

> 6 type — workload + cost / performance. gp3 default + io2 high-end + Block Express premium + st1/sc1 throughput HDD. EC2 의 [[../../../compute/ec2/03-storage/01-ebs-deep]] 와 cross-reference.

## 1. 6 type 요약

| Type | Family | Max Size | Max IOPS | Max Throughput | $/GB·월 | Min duration |
| --- | --- | --- | --- | --- | --- | --- |
| **gp3** | SSD | 16 TiB | 16K (3K free) | 1000 MB/s (125 free) | $0.096 | none |
| **gp2** | SSD (legacy) | 16 TiB | 16K | size 따라 | $0.114 | none |
| **io2** | SSD | 64 TiB | 256K | 4000 MB/s | $0.142 + IOPS | none |
| **io2 Block Express** | SSD premium | 64 TiB | 256K | 4000 MB/s | premium | none |
| **st1** | HDD | 16 TiB | low (250 baseline) | 500 MB/s burst | $0.054 | none |
| **sc1** | HDD | 16 TiB | low | 250 MB/s | $0.018 | none |

> Boot volume = gp3 / gp2 / io2 / io1 만. st1/sc1 = data volume.

## 2. gp3 — modern default

```
baseline (size 무관, 무료):
  3000 IOPS + 125 MB/s

provisioned extra:
  IOPS:        +$0.005/IOPS·월 (3000 이상)
  Throughput:  +$0.04/MB·s·월 (125 이상)
  max:         16000 IOPS + 1000 MB/s
```

gp2 와 비교:
| size | gp2 free IOPS | gp3 free IOPS |
| --- | --- | --- |
| 50 GB | 150 | 3000 |
| 100 GB | 300 | 3000 |
| 1 TB | 3000 | 3000 |

→ **작은 volume 일수록 gp3 가 압도적**.

## 3. gp2 → gp3 migration

```bash
aws ec2 modify-volume --volume-id vol-xxx --volume-type gp3
```

zero-downtime, 6h 안 한번. 비용도 ~20% 절감.

## 4. io2 — high-performance DB

| | io1 (legacy) | io2 |
| --- | --- | --- |
| Durability | 99.8-99.9% | 99.999% |
| IOPS:GB ratio | 50:1 | 500:1 |
| Max IOPS | 64K | 256K (only on Nitro instances) |
| Multi-Attach | ✓ (Nitro 만) | ✓ |
| sub-ms latency | ✗ | ✓ |

가격:
```
$0.142/GB + $0.065/IOPS·월
```

활용:
- Oracle / SQL Server / Postgres / MySQL primary
- SAP HANA (Block Express 권장)
- Multi-Attach (cluster DB)

## 5. io2 Block Express

```
Max IOPS:        256K (io2 와 동일하지만 throughput / latency 개선)
Max Throughput:  4000 MB/s
Latency:         500μs sub-ms
Size:            64 TiB
```

지원 instance: R5b, R6in, X2idn, M6in, M6i (제한적).

## 6. st1 — Throughput Optimized HDD

```
baseline:  40 MB/s/TiB
burst:     up to 500 MB/s
```

활용:
- Big data sequential read (EMR / Hadoop)
- Log aggregation
- Streaming workload
- Data warehouse

> Random IO 안 됨. Boot 불가.

## 7. sc1 — Cold HDD

```
baseline:  12 MB/s/TiB
burst:     up to 250 MB/s
```

활용:
- Cold data archive
- Infrequent access
- 비용 최저 ($0.018/GB·월)

## 8. Volume 생성 / modify

### CLI
```bash
aws ec2 create-volume --availability-zone ap-northeast-2a \
  --size 100 \
  --volume-type gp3 \
  --iops 5000 \
  --throughput 250 \
  --encrypted \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=app-data}]'

# Modify (zero-downtime)
aws ec2 modify-volume --volume-id vol-xxx --size 200 --iops 8000 --throughput 500
```

| Modify 한도 |
| --- |
| 6h 마다 1회 |
| 축소 불가 (snapshot 후 작은 volume 재생성) |
| 처리 ~수분 |

## 9. Attach / Detach

```bash
aws ec2 attach-volume --volume-id vol-xxx --instance-id i-xxx \
  --device /dev/sdf

aws ec2 detach-volume --volume-id vol-xxx
```

> /dev/sdf 같은 device name 은 hint — 실제 NVMe 는 /dev/nvme1n1 등.

```bash
# Instance 안에서
lsblk
sudo mkfs -t xfs /dev/nvme1n1
sudo mkdir /data && sudo mount /dev/nvme1n1 /data
echo '/dev/nvme1n1 /data xfs defaults,nofail 0 2' | sudo tee -a /etc/fstab
```

## 10. NVMe 의 모든 modern instance

Nitro instance (m5+, c5+, r5+, t3+, etc) = EBS device 가 NVMe block device. 옛 BSD/Xen-style `/dev/xvdf` 가 아닌 `/dev/nvme1n1` 등.

## 11. EBS-Optimized

Nitro / 7세대+ = 자동 활성. EBS bandwidth 가 network 와 분리.

```
m7g.large:   EBS 최대 10 Gbps / 펌 1 Gbps
m7g.16xlarge: EBS 50 Gbps
```

## 12. Boot vs Data Volume

| | Boot (root) | Data |
| --- | --- | --- |
| Type | gp3 / gp2 / io2 / io1 | 모든 type |
| DeleteOnTermination | true (default) | false (default) |
| Size | OS 최소 + 여유 (30 GB) | workload 따라 |
| Snapshot for AMI | ✓ | optional |

## 13. AZ scope — 이동

```bash
# Snapshot → 다른 AZ
aws ec2 create-snapshot --volume-id vol-xxx

aws ec2 create-volume --snapshot-id snap-xxx \
  --availability-zone ap-northeast-2c --volume-type gp3
```

## 14. CDK / Terraform

```typescript
new ec2.Volume(this, 'Data', {
  availabilityZone: 'ap-northeast-2a',
  size: cdk.Size.gibibytes(100),
  volumeType: ec2.EbsDeviceVolumeType.GP3,
  iops: 5000,
  throughput: 250,
  encrypted: true,
});
```

```hcl
resource "aws_ebs_volume" "data" {
  availability_zone = "ap-northeast-2a"
  size              = 100
  type              = "gp3"
  iops              = 5000
  throughput        = 250
  encrypted         = true
  tags              = { Name = "app-data" }
}
```

## 15. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| gp2 그대로 사용 | gp3 migration |
| io2 IOPS 필요 없는 데 비싸게 | gp3 충분 |
| size 줄이려고 시도 | 축소 불가, snapshot 으로 |
| Cross-AZ attach 시도 | snapshot |
| Block Express + 비-지원 instance | R5b 등만 |
| st1 random IO | sequential 만 |

## 16. 다음

- ▶ [[02-snapshot-and-ami]]
- ▶ [[03-pricing-and-limits]]
- ▶ [[../02-operations/01-encryption-kms]]

## 17. 영상 / 자료

| 자료 |
| --- |
| [EBS Volume Types](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-volume-types.html) |
| [io2 Block Express](https://aws.amazon.com/blogs/aws/amazon-ebs-block-express-volumes-with-amazon-ec2-r5b-instances-are-now-generally-available/) |
| [gp3 발표](https://aws.amazon.com/blogs/aws/new-amazon-ebs-gp3-volume-lets-you-provision-performance-apart-from-capacity/) |

## 18. 참고

- [[../ebs]]
- [[../../../compute/ec2/03-storage/01-ebs-deep]]
- [[../../../compute/ec2/01-fundamentals/02-instance-types]] §EBS-Optimized
