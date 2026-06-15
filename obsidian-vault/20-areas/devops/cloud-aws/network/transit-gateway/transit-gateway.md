---
title: "AWS Transit Gateway"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:53:00+09:00
tags:
  - aws
  - network
  - tgw
  - hybrid
---

# AWS Transit Gateway (TGW)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Transit Gateway hub |

**[[../network|↑ Network]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

여러 VPC + 온프레 VPN + Direct Connect 를 **허브 (hub-and-spoke) 로** 묶는 라우터. VPC peering 의 n² 폭발 문제를 해결.

## 2. 왜 / 언제

- VPC 가 10개 이상 — peering 그리드는 관리 불가
- 멀티 account + 멀티 region 네트워크 통합
- 온프레 VPN / Direct Connect 를 한 곳에서 종단
- shared services VPC (DNS / Active Directory / ECR) 를 모두에게
- 라우팅 정책 / 격리 (route table 으로)

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Attachment** | VPC / VPN / Direct Connect / Peering / Connect 를 TGW 에 붙임 |
| **Route Table** | attachment 별로 다른 라우팅 가능 (격리) |
| **Propagation** | attachment 가 자기 CIDR 을 TGW route table 에 자동 등록 |
| **Association** | attachment 가 어떤 route table 을 쓸지 |
| **TGW Peering** | region 간 TGW 연결 |
| **Connect** | SD-WAN appliance 와 GRE / BGP 연결 |
| **Network Manager** | 전체 토폴로지 view |

## 4. 표준 토폴로지

```
                  Transit Gateway
       ┌────────────┼────────────┐
       │            │            │
   VPC-shared    VPC-prod     VPC-dev
   (DNS / ECR)
       │
   on-prem (VPN / DX)
```

- shared 의 RT 는 모든 spoke 의 CIDR 추가
- prod RT 는 shared + on-prem 만 (dev 차단 = 격리)
- 응용 VPC 끼리는 default 차단

## 5. TGW vs VPC Peering vs PrivateLink

| | TGW | VPC Peering | PrivateLink |
| --- | --- | --- | --- |
| 패턴 | hub-and-spoke | 1:1 mesh | endpoint 1개 노출 |
| 라우팅 | 중앙 | 양쪽 RT 모두 | endpoint |
| transitive | ✅ | X | n/a |
| 비용 | TGW 시간 + GB | 무료 (cross-AZ DT 있음) | endpoint 시간 + GB |
| 적합 | 다수 VPC | 단순 2개 | 서비스 노출 |

## 6. 비용 / 함정

- TGW 시간당 ~$0.05 + attachment 시간당 ~$0.05 + GB-processed $0.02
- **GB 처리 비용 누적** 큼 — large traffic 은 비싸짐
- 라우트 너무 많아지면 (5000+) BGP propagation 느려짐
- **격리 위해 multi route table** — 처음부터 설계
- cross-region TGW peering 도 가능 (region 간 backbone)

## 7. 관련

- [[../network|↑ Network]]
- [[../vpc/vpc]]
- [[../privatelink/privatelink]]
- [[../../management/organizations/organizations]] — RAM 으로 cross-account TGW 공유
