---
title: "AWS EBS — Elastic Block Store"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:35:00+09:00
tags:
  - aws
  - storage
  - ebs
---

# AWS EBS

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | EBS 개념 + 사용 |

**[[storage|↑ Storage]]** · **[[cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**EC2 의 가상 디스크**. 같은 AZ 의 EC2 에 attach → OS 가 일반 디스크처럼 마운트.

---

## 2. 왜

- **EC2 의 root / data disk**
- **multi-AZ snapshot → 재해 복구**
- **NVMe 수준 IOPS / throughput**
- **수정 가능** (크기 / 종류 변경 — 서버 켜둔 채로)

대안:
- **Instance Store** (NVMe local) — 빠르지만 stop 시 사라짐
- **EFS / FSx** — NFS / SMB — 여러 인스턴스 공유
- **S3** — Object — 디스크 X

---

## 3. Volume Type

| Type | 용도 | IOPS | Throughput |
| --- | --- | --- | --- |
| **gp3** (권장) | 범용 SSD | 3K (max 16K) | 125 MB/s (max 1000) |
| **gp2** | 옛 범용 SSD | size × 3 | size 비례 |
| **io2 / io2 Block Express** | 최고 IOPS | 256K | 4 GB/s |
| **st1** | throughput 최적 HDD | - | 500 MB/s |
| **sc1** | cold HDD | - | 250 MB/s |

→ 대부분 **gp3**. 큰 DB / latency 민감 = io2.

---

## 4. 가격 (Seoul)

| Type | $/GB·월 | 추가 |
| --- | --- | --- |
| gp3 | $0.0912 | IOPS 3K free / throughput 125 MB/s free |
| gp2 | $0.114 | IOPS size 비례 |
| io2 | $0.125 + IOPS $0.065 | |
| st1 | $0.045 | |
| sc1 | $0.015 | |
| Snapshot | $0.05/GB·월 | |

→ gp3 가 gp2 보다 싸고 더 좋음. 마이그.

---

## 5. 설치 / 사용

### 5.1 새 volume 생성 + attach

```bash
# 생성
aws ec2 create-volume \
  --availability-zone ap-northeast-2a \
  --size 100 \
  --volume-type gp3 \
  --iops 3000 \
  --throughput 125

# attach
aws ec2 attach-volume --volume-id vol-xxx --instance-id i-xxx --device /dev/sdf
```

### 5.2 인스턴스 안에서 (Linux)

```bash
# block device 확인
lsblk
# /dev/nvme1n1 (보통 nvme 로 보임, /dev/sdf 가 아님)

# 파일 시스템
sudo mkfs -t xfs /dev/nvme1n1

# mount
sudo mkdir /data
sudo mount /dev/nvme1n1 /data

# 영구 mount (재부팅 후)
sudo blkid /dev/nvme1n1                            # UUID
sudo vim /etc/fstab
# UUID=xxx-xxx /data xfs defaults,nofail 0 2
```

### 5.3 크기 증가 (online)

```bash
# 1. 볼륨 크기 증가 (AWS)
aws ec2 modify-volume --volume-id vol-xxx --size 200 --volume-type gp3

# 2. OS 파일 시스템 확장
sudo growpart /dev/nvme1n1 1                      # partition (있다면)
sudo xfs_growfs /data                              # XFS
# 또는 ext4: sudo resize2fs /dev/nvme1n1
```

→ 다운타임 X.

### 5.4 Terraform

```hcl
resource "aws_ebs_volume" "data" {
  availability_zone = "ap-northeast-2a"
  size              = 100
  type              = "gp3"
  iops              = 3000
  throughput        = 125
  encrypted         = true
  kms_key_id        = aws_kms_key.ebs.arn
}

resource "aws_volume_attachment" "data" {
  device_name = "/dev/sdf"
  volume_id   = aws_ebs_volume.data.id
  instance_id = aws_instance.app.id
}
```

---

## 6. Snapshot

```bash
# 생성
aws ec2 create-snapshot --volume-id vol-xxx --description "daily"

# 자동 (Data Lifecycle Manager — DLM)
aws dlm create-lifecycle-policy ...

# 복원
aws ec2 create-volume --snapshot-id snap-xxx --availability-zone ap-northeast-2a
```

특징:
- incremental — 변경 block 만
- S3 에 저장 (region durable)
- region 간 copy 가능 (DR)
- 암호화 (KMS)

→ DLM 으로 매일 자동 + retention.

---

## 7. 암호화

```bash
# 새 볼륨 — Encrypted=true
aws ec2 create-volume --encrypted --kms-key-id alias/aws/ebs ...

# 기본 활성 (region 별)
aws ec2 enable-ebs-encryption-by-default
```

암호화된 볼륨에서 snapshot → 암호화된 snapshot. 옛 unencrypted 마이그 = snapshot → copy with encryption.

---

## 8. Multi-Attach (io2 만)

```
한 io2 볼륨을 16 개 EC2 에 동시 attach (같은 AZ)
→ 클러스터 파일시스템 (GFS, OCFS2) 또는 raw shared
```

흔한 케이스 X. 일반 = file system 공유는 EFS.

---

## 9. 성능 — 분석

```bash
# 인스턴스 안
iostat -x 1
fio --name=test --size=1G --bs=4k --rw=randread --iodepth=32

# CloudWatch
VolumeReadOps, VolumeWriteOps
VolumeQueueLength
VolumeIdleTime
BurstBalance (gp2)
```

gp3 의 baseline 3K IOPS / 125 MB/s 가 부족 → 직접 추가.

---

## 10. 사용 시나리오

- EC2 root volume (필수)
- DB self-managed
- 빌드 / artifact / cache
- 큰 file 처리 임시
- container host 의 docker storage

부적합:
- 여러 인스턴스 공유 → EFS
- 글로벌 / public → S3

---

## 11. 함정

### 11.1 stop / start 후 device name
`/dev/sdf` 가 `/dev/nvme1n1` 로 보임. fstab 은 UUID 권장.

### 11.2 데이터 손실 방지
인스턴스 termination 시 root volume = 삭제 default. **Data volume** 은 `DeleteOnTermination=false` 명시.

### 11.3 같은 AZ 만
EBS 는 single-AZ. 다른 AZ 로는 snapshot 통해 copy.

### 11.4 gp2 의 burst balance
작은 (< 1 TB) gp2 = burst credit. 고갈 시 baseline 으로. gp3 에선 X.

### 11.5 IOPS 추가의 throttle
요청 후 일정 시간 대기 (modify-volume).

### 11.6 snapshot 비용 누적
정기 만료 정책.

### 11.7 암호화 키 분실
KMS key 삭제 = 볼륨 영구 손실. 신중.

---

## 12. 학습 자료

- AWS EBS docs
- **EBS Performance** AWS blog
- `iostat` / `fio` 기본

---

## 13. 관련

- [[storage]] — Storage hub
- [[s3]] — Object 대안
- [[ec2]] — EBS 의 user
- [[kms]] — 암호화
