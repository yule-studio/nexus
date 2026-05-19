---
title: "AWS EC2 — Elastic Compute Cloud"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:15:00+09:00
tags:
  - aws
  - compute
  - ec2
---

# AWS EC2

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | EC2 개념 + 설치 + 사용 |

**[[compute|↑ Compute]]** · **[[cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**가상 머신 (VM)** — 임의의 OS / 응용 실행. AWS 의 가장 기본 자원.

---

## 2. 왜

- **on-premise 서버 단점** — 사전 구매 / 용량 추정 어려움 / 유지보수
- 클라우드 VM = 분 단위 과금 / 즉시 생성·삭제 / API
- 정형화된 응용 / 컨테이너 / DB / 빌드 머신 / VPN 등 어디서나 base

대안:
- **ECS / EKS** — container 표준 — VM 직접 관리 안 함
- **Lambda** — serverless — 짧은 작업
- **Lightsail** — 작은 site (월정액)

---

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Instance Type** | CPU/RAM 종류 (`t3.medium`, `c6i.large`, `r6i.xlarge`) |
| **AMI** | Amazon Machine Image — OS + pre-install snapshot |
| **EBS Volume** | 디스크 ([[ebs]]) |
| **Key Pair** | SSH key (생성 시 1회만 download) |
| **Security Group** | 인스턴스 방화벽 (stateful) |
| **VPC / Subnet** | 네트워크 ([[vpc]]) |
| **Elastic IP** | 고정 public IP |
| **Auto Scaling Group (ASG)** | 부하 따라 수량 ↑↓ |
| **Launch Template** | 인스턴스 생성 템플릿 |

---

## 4. Instance Family

| Family | 용도 | 예 |
| --- | --- | --- |
| **t** | 범용 (burstable) | t3.medium |
| **m** | 범용 (steady) | m6i.large |
| **c** | CPU intensive | c6i.large |
| **r** | RAM 많음 | r6i.large |
| **i / d** | NVMe local SSD | i4i.large |
| **g / p** | GPU | g5.xlarge, p4d.24xlarge |
| **x** | 거대 RAM (1-24 TB) | x2idn.16xlarge |
| **a** | ARM (Graviton) | m7g.large |

**Graviton (ARM)** = x86 대비 20-40% 더 빠르고 싸다. 호환되면 우선 검토.

---

## 5. 설치 / 첫 인스턴스

### 5.1 Console (GUI)

```
EC2 → Launch Instance
  Name:                myapp
  AMI:                  Amazon Linux 2023 (또는 Ubuntu 22.04 LTS)
  Instance type:         t3.medium
  Key pair:              새로 생성 → myapp.pem 다운로드
  Network:               VPC + Subnet (public for SSH)
  Security Group:        SSH (22) 내 IP 만
  Storage:               gp3 30 GB
Launch
```

### 5.2 CLI

```bash
aws ec2 run-instances \
  --image-id ami-0c9c942bd7bf113a2 \
  --instance-type t3.medium \
  --key-name myapp \
  --security-group-ids sg-xxx \
  --subnet-id subnet-xxx \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=myapp}]'

aws ec2 describe-instances --instance-ids i-xxx
aws ec2 stop-instances --instance-ids i-xxx
aws ec2 start-instances --instance-ids i-xxx
aws ec2 terminate-instances --instance-ids i-xxx
```

### 5.3 Terraform

```hcl
resource "aws_instance" "myapp" {
  ami           = "ami-0c9c942bd7bf113a2"
  instance_type = "t3.medium"
  key_name      = aws_key_pair.myapp.key_name
  subnet_id     = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]

  root_block_device {
    volume_size = 30
    volume_type = "gp3"
  }

  tags = { Name = "myapp" }
}
```

---

## 6. 접속

```bash
chmod 400 myapp.pem
ssh -i myapp.pem ec2-user@<public-ip>            # Amazon Linux
ssh -i myapp.pem ubuntu@<public-ip>               # Ubuntu

# Session Manager (key 없이)
aws ssm start-session --target i-xxx
```

Session Manager 가 권장 — SSH 22 포트 안 열어도 됨, IAM 권한.

---

## 7. User Data (boot script)

```bash
#!/bin/bash
yum update -y
yum install -y docker
systemctl enable --now docker
```

`user_data` 로 인스턴스 시작 시 자동 실행. cloud-init 표준.

---

## 8. EBS — 디스크

자세히 → [[ebs]]

```bash
# 추가 볼륨 attach
aws ec2 attach-volume --volume-id vol-xxx --instance-id i-xxx --device /dev/sdf

# 인스턴스 안에서
lsblk
sudo mkfs -t xfs /dev/nvme1n1
sudo mkdir /data && sudo mount /dev/nvme1n1 /data
```

`gp3` (기본) — 4 TB / 16K IOPS / 1000 MB/s 까지 free. 추가 IOPS / throughput 별도 과금.

---

## 9. Auto Scaling

```hcl
resource "aws_launch_template" "app" { ... }

resource "aws_autoscaling_group" "app" {
  min_size            = 2
  max_size            = 10
  desired_capacity    = 2
  vpc_zone_identifier = aws_subnet.private[*].id
  launch_template { id = aws_launch_template.app.id; version = "$Latest" }
  target_group_arns   = [aws_lb_target_group.app.arn]
}

resource "aws_autoscaling_policy" "cpu" {
  name                   = "cpu-target"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "TargetTrackingScaling"
  target_tracking_configuration {
    predefined_metric_specification { predefined_metric_type = "ASGAverageCPUUtilization" }
    target_value = 60.0
  }
}
```

→ CPU 60% 목표로 자동 조정.

---

## 10. 가격 모델

| 모델 | 할인 |
| --- | --- |
| On-Demand | 0% |
| Savings Plan / RI 1년 | 30-40% |
| Savings Plan / RI 3년 | 50-70% |
| Spot | 최대 90% (강제 종료 위험) |

운영 = Savings Plan + Spot 혼합. 항상 켜둘 자원 = SP/RI, 일시·내결함 = Spot.

---

## 11. 비용 (Seoul region 예)

| 인스턴스 | $/시간 | $/월 (24x7) |
| --- | --- | --- |
| t3.micro | $0.0130 | ~$9.5 |
| t3.medium | $0.0520 | ~$38 |
| m6i.large | $0.107 | ~$78 |
| c6i.large | $0.096 | ~$70 |
| r6i.large | $0.151 | ~$110 |
| m7g.large (Graviton) | $0.0856 | ~$62 |

+ EBS / data transfer.

---

## 12. 사용 시나리오

- **웹 서버** (Nginx / Spring / Node)
- **DB self-managed** (PostgreSQL 직접 — 운영 부담 ↑)
- **CI runner** — GitHub Actions self-hosted
- **VPN bastion** (jump host)
- **개발 환경** — 큰 RAM / GPU 필요 시
- **Container host** — ECS EC2 launch (드물게)

⚠️ container 가 표준 시 = EKS / ECS Fargate 우선. EC2 는 비-container 작업.

---

## 13. 함정

### 13.1 큰 인스턴스를 24x7
필요 시간만. Auto Scaling / 야간 stop.

### 13.2 Public IP 직노출
SSH 22 인터넷 노출 → 매일 brute force. Session Manager / Bastion / VPN.

### 13.3 Spot interruption
2 분 통지 후 강제 종료. stateless 워크로드만.

### 13.4 EBS gp2 vs gp3
gp3 가 더 싸고 빠름. 마이그.

### 13.5 인스턴스 store (NVMe local)
stop 하면 데이터 사라짐. EBS 다름.

### 13.6 OS 패치 / 보안
Systems Manager Patch Manager — 자동.

---

## 14. 학습 자료

- AWS EC2 docs
- AWS Well-Architected — Compute pillar
- **AWS Skill Builder — EC2 fundamentals**

---

## 15. 관련

- [[compute]] — Compute hub
- [[ecs]] — Container 대안
- [[lambda]] — Serverless 대안
- [[ebs]] — 디스크
- [[vpc]] — 네트워크
- [[iam]] — IAM role
