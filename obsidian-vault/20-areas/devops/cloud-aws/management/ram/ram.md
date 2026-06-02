---
title: "AWS Resource Access Manager (RAM)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:31:00+09:00
tags:
  - aws
  - management
  - cross-account
  - ram
---

# AWS Resource Access Manager (RAM)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | RAM hub |

**[[../management|↑ Management]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

자원을 **다른 AWS account 와 공유** (복제 X). Subnet / Transit Gateway / License / Route 53 Resolver / RDS snapshot 등.

## 2. 왜 / 언제

- 멀티 account 에 공통 VPC subnet (네트워크 중앙 관리)
- TGW / DXGW 를 모든 account 가 사용
- Aurora / RDS snapshot 을 다른 account 에 복원
- AMI / License Configuration cross-account 공유

## 3. 함정

- 무료 (RAM 자체)
- Organizations 위에 있으면 share principal 로 OU 지정 가능
- 받는 쪽 account 는 "ResourceShareInvitationArn" 수락
- 공유 자원은 받는 쪽에서 IAM 권한 따로

## 4. 관련

- [[../management|↑ Management]]
- [[../organizations/organizations]]
- [[../../network/transit-gateway/transit-gateway]]
- [[../../network/vpc/vpc]]
