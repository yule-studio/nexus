---
title: "EC2 01-2 — Instance Types (7 family + Graviton)"
kind: knowledge
status: current
created_at: 2026-05-22T15:04:00+09:00
tags: [aws, ec2, fundamentals, instance-type, graviton]
home_hub: ec2
---

# EC2 Instance Types

> Family + Generation + Capability + Size. 800+ type 의 분류 체계 이해가 right-sizing 의 시작.

## 1. 이름 규칙

```
m  6  i  d  n  .  large
│  │  │  │  │     │
│  │  │  │  │     └─ size (nano/micro/.../48xlarge)
│  │  │  │  └─────── n: 100Gbps network
│  │  │  └────────── d: NVMe local SSD
│  │  └───────────── i: Intel  /  a: AMD  /  g: Graviton (ARM)
│  └──────────────── generation (6 = gen 6)
└─────────────────── family
```

예시:
- `m6i.large` — 범용 6세대 Intel
- `m7g.large` — 범용 7세대 Graviton (ARM)
- `c6gn.4xlarge` — 컴퓨팅 6세대 Graviton + 100Gbps
- `r7iz.metal-32xl` — 메모리 7세대 Intel SapphireRapids bare metal

## 2. Family table

### General Purpose
| Family | 특징 | 예 |
| --- | --- | --- |
| **t** (3/4g) | burstable CPU credit | t3.medium |
| **m** (6/7) | balanced CPU/RAM 1:4 | m6i.large / m7g.large |
| **mac** | macOS bare metal | mac2.metal |

### Compute Optimized
| | | |
| --- | --- | --- |
| **c** (6/7) | CPU 강세 1:2 | c6i.large / c7g.large |
| **c7i-flex** | flex (성능 절약) | c7i-flex.large |

### Memory Optimized
| | | |
| --- | --- | --- |
| **r** (6/7) | RAM 1:8 | r6i.large |
| **x** (2idn) | RAM 1:32, 4TB+ | x2idn.16xlarge |
| **u** | 6-24TB RAM (SAP HANA) | u-9tb1.112xlarge |
| **z1d** | high frequency + RAM | z1d.large |

### Storage Optimized
| | | |
| --- | --- | --- |
| **i** (3/4i) | NVMe local | i4i.large |
| **d** (3) | dense HDD | d3.xlarge |
| **im4gn / is4gen** | Graviton + NVMe | im4gn.large |

### Accelerated
| | | |
| --- | --- | --- |
| **p** (3/4/5) | NVIDIA GPU (training) | p5.48xlarge (H100) |
| **g** (5/6) | NVIDIA GPU (inference) | g5.xlarge (A10G) |
| **dl** (1) | Habana Gaudi | dl1.24xlarge |
| **inf** (2) | AWS Inferentia | inf2.xlarge |
| **trn** (1) | AWS Trainium | trn1.32xlarge |
| **f** (1/2) | FPGA | f1.2xlarge |

### HPC
| | | |
| --- | --- | --- |
| **hpc** (6/7) | EFA 400Gbps | hpc7g.16xlarge |

## 3. Generation = 새로움

| Gen | CPU | 특징 |
| --- | --- | --- |
| 4 | 2019 | Skylake / Cascade |
| 5 | 2020 | Cascade |
| 6 | 2021 | Ice Lake (Intel) / Milan (AMD) / Graviton2 |
| 7 | 2023 | Sapphire Rapids (Intel) / Genoa (AMD) / Graviton3 |

> 같은 가격이면 **항상 최신 gen 사용**. 7세대가 5세대보다 30~40% 빠르고 같은 가격.

## 4. Burstable t-class

t3/t4g 는 **CPU credit** 기반:
```
baseline 20% + burst 100%
사용 안 한 시간 → credit 축적 (24h cap)
credit 다 떨어지면 standard mode 면 baseline 강제, unlimited mode 면 $0.05/vCPU·hour 추가
```

```bash
aws ec2 modify-instance-credit-specification --instance-credit-specification "InstanceId=i-xxx,CpuCredits=unlimited"
```

> production = `unlimited` 권장. 갑작스러운 traffic spike 안전.

## 5. Graviton (ARM64) — 우선 검토

| | x86 (m6i) | Graviton (m7g) |
| --- | --- | --- |
| 가격 | 100% | ~80% |
| 성능 | 100% | 110~140% |
| 가성비 | 1.0x | 1.5~2.0x |

| 호환 |
| --- |
| ✓ Java / Python / Node / Go / Rust |
| ✓ Spring Boot / nginx / PostgreSQL / MySQL / Redis |
| △ Docker image 가 multi-arch 빌드돼야 함 |
| ✗ 일부 native extension / 일부 oracle / 일부 windows |

bkurs-be: Java 21 + Spring Boot 3.5 = ARM64 완전 호환 → **m7g / c7g / r7g 권장**.

## 6. Sizing

| 사이즈 | vCPU | RAM (m-class) |
| --- | --- | --- |
| nano | 2 (burstable) | 0.5 GB |
| micro | 2 (burstable) | 1 GB |
| small | 2 (burstable) | 2 GB |
| medium | 2 | 4 GB |
| large | 2 | 8 GB |
| xlarge | 4 | 16 GB |
| 2xlarge | 8 | 32 GB |
| 4xlarge | 16 | 64 GB |
| ...48xlarge | 192 | 768 GB |
| metal | full host | full host RAM |

## 7. Right-sizing 방법

```bash
# Compute Optimizer
aws compute-optimizer get-ec2-instance-recommendations \
  --instance-arns arn:aws:ec2:...:instance/i-xxx
```

권장 = CPU avg ≤ 40% + Mem avg ≤ 60% → 한 단계 down.

## 8. EBS-Optimized

7세대 이상 = 기본 활성. EBS bandwidth 가 network 와 분리.

```
m7g.large: EBS 최대 10 Gbps / 펌 1 Gbps
m7g.16xlarge: EBS 50 Gbps
```

## 9. Network Performance

| 표시 | 의미 |
| --- | --- |
| Low / Moderate | best-effort |
| Up to 10/25 Gigabit | burst |
| 25 Gigabit | sustained |
| 50/100 Gigabit | premium (n suffix) |
| 200/400 Gbps (EFA) | HPC only |

## 10. Bare Metal

`.metal` suffix — hypervisor 없음, 직접 host. 라이센스 BYOL (Oracle / VMware) 또는 nested virtualization.

## 11. Capacity Reservation

```bash
aws ec2 create-capacity-reservation \
  --instance-type m7g.large --instance-platform "Linux/UNIX" \
  --availability-zone ap-northeast-2a --instance-count 5 \
  --end-date-type unlimited
```

→ 특정 AZ 에 capacity 보장. 가격은 On-Demand 와 동일.

## 12. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| t3 standard mode burst 고갈 | unlimited |
| x86 만 사용 (Graviton 무시) | multi-arch build |
| 5세대 그대로 | 7세대 마이그 |
| over-provision | Compute Optimizer |
| Bare metal 비용 모름 | EC2 metal 만 |

## 13. 다음

- ▶ [[03-pricing-models]]
- ▶ [[04-amis-and-bootstrap]]
- ▶ [[../03-storage/01-ebs-deep]] — EBS-Optimized

## 14. 영상 / 자료

| 자료 |
| --- |
| [Graviton — AWS](https://aws.amazon.com/ec2/graviton/) |
| [Instance Type 공식](https://aws.amazon.com/ec2/instance-types/) |
| [Compute Optimizer](https://aws.amazon.com/compute-optimizer/) |

## 15. 참고

- [[../ec2]]
- [[../03-storage/01-ebs-deep]]
- [[../../../../../docker/practice/06-multi-platform-build]]
