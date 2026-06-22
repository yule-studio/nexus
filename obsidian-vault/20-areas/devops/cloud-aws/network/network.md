---
title: "AWS Network (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:08:00+09:00
tags:
  - aws
  - network
  - hub
---

# AWS Network (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 카테고리 hub |

**[[../cloud-aws|↑ AWS]]**

---

## 1. 서비스

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **VPC** | [[vpc]] | 가상 네트워크 — 모든 자원의 base |
| **ALB / NLB** | [[alb]] | Load Balancer (L7 / L4) |
| **CloudFront** | [[cloudfront]] | CDN — 전 세계 edge cache |
| **Route 53** | [[route53]] | DNS + health check + traffic policy |

### 1.1 연결·하이브리드

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Direct Connect** | [[direct-connect/direct-connect\|direct-connect]] | 데이터센터 ↔ AWS 전용 회선 (1G~100G) — 안정 latency + 저비용 데이터 전송 |
| **VPN** | [[vpn/vpn\|vpn]] | Site-to-Site (온프레 ↔ AWS) + Client VPN (개인 디바이스 ↔ VPC) — IPsec 기반 연결 |
| **Transit Gateway** | [[transit-gateway/transit-gateway\|transit-gateway]] | 여러 VPC + 온프레 VPN + Direct Connect 를 묶는 허브 라우터 — VPC peering 폭발 해결 |
| **PrivateLink** | [[privatelink/privatelink\|privatelink]] | 인터넷 / NAT 없이 다른 VPC / SaaS 서비스에 사설 endpoint 로 접근 |

### 1.2 가속·엣지

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Global Accelerator** | [[global-accelerator/global-accelerator\|global-accelerator]] | Anycast IP + AWS 백본 — 비-HTTP (TCP / UDP / 게임) 글로벌 latency 단축 + failover |
| **NLB** | [[nlb/nlb\|nlb]] | L4 (TCP / UDP / TLS) 로드밸런서 — 정적 IP / 초고속 RPS / 저latency, PrivateLink provider 역할 |

### 1.3 디스커버리·API

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Cloud Map** | [[cloud-map/cloud-map\|cloud-map]] | 서비스 디스커버리 — 응용이 다른 서비스의 endpoint 를 DNS / API 로 동적 조회 |
| **API Gateway** | [[api-gateway/api-gateway\|api-gateway]] | 매니지드 HTTP / REST / WebSocket API 게이트웨이 — Lambda / ALB 앞에서 인증 / throttling / 캐싱 |

---

## 2. 토폴로지 단순 예

```
인터넷
  ↓
Route 53 (DNS)
  ↓
CloudFront (CDN) — 정적
  ↓
ALB (load balancer)
  ↓
VPC public subnet — NAT GW
       │
       └─ private subnet — EC2 / ECS / RDS
```

---

## 3. 비용 함정

- **NAT Gateway** — 매우 비쌈 ($32/월 + GB-out)
- **Data Transfer Out** — 인터넷으로 나가는 트래픽
- **VPC Endpoint** — S3 / DynamoDB 트래픽이 NAT 안 거치게
- **PrivateLink** — 다른 VPC 의 서비스에 private 접근

---

## 4. 관련

- [[../cloud-aws|↑ AWS]]
- [[../security/iam]] — Security Group 도 일부
- [[../../../computer-science/network/network|↗ network 이론]]
