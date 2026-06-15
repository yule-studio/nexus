---
title: "Amazon Lightsail"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:13:00+09:00
tags:
  - aws
  - compute
  - vps
---

# Amazon Lightsail

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Lightsail hub |

**[[../compute|↑ Compute]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

EC2 + RDS + LB + 스토리지 + 트래픽을 **정액제로 묶은** 단순 VPS. DigitalOcean / Linode 의 AWS 대응. 개인 / 소규모 프로젝트용.

## 2. 왜 / 언제

- 개인 블로그 / 사이드 프로젝트 (월 $3.5~ 부터)
- WordPress / Magento / Plesk 의 한 번 클릭 설치
- 비용 예측 가능성 (정액제)
- VPC / IAM 복잡함 회피

## 3. 핵심 / 함정

| 개념 | 의미 |
| --- | --- |
| **Instance** | EC2 미니 (Ubuntu / Amazon Linux / 앱 마켓플레이스) |
| **Database** | 매니지드 MySQL / Postgres 미니 |
| **Container Service** | Lightsail 안의 컨테이너 (간단) |
| **Bundle** | 정액제 묶음 (메모리 / CPU / 스토리지 / 트래픽 포함) |
| **함정** | AWS 본격 자원 (VPC peering 등) 과 통합 한정. 성장하면 EC2 로 마이그 |

## 4. 관련

- [[../compute|↑ Compute]]
- [[../ec2/ec2]] — 본격 운영
