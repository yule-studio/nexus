---
title: "AWS Direct Connect"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:24:00+09:00
tags:
  - aws
  - network
  - hybrid
  - direct-connect
---

# AWS Direct Connect (DX)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Direct Connect hub |

**[[../network|↑ Network]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

데이터센터 ↔ AWS **전용 회선** (인터넷 안 거침). 대역폭 1G ~ 100G, 안정 latency, 데이터 전송 비용 저렴.

## 2. 왜 / 언제

- VPN 의 latency / 변동성 불충분 (금융 / 게임 / DB replication)
- 대용량 트래픽 (PB 마이그레이션, 분당 GB 백업)
- 컴플라이언스 (트래픽이 인터넷을 안 지나가야)
- AWS data transfer out 비용이 큰 워크로드

## 3. 핵심 / 함정

| 개념 | 의미 |
| --- | --- |
| **Dedicated Connection** | 단독 1G / 10G / 100G 회선 |
| **Hosted Connection** | 파트너가 공유 회선 일부 (50M ~ 10G) |
| **Public VIF** | S3 / DynamoDB 등 public AWS endpoint |
| **Private VIF** | VPC 안 자원 |
| **Transit VIF** | Transit Gateway 와 연결 |
| **DXGW** | 여러 region VPC 와 한 DX 연결 |
| **함정** | 신청 → 회선 까는 데 수 주 ~ 수 개월. 백업 VPN 같이 |

## 4. 관련

- [[../network|↑ Network]]
- [[../vpn/vpn]]
- [[../transit-gateway/transit-gateway]]
