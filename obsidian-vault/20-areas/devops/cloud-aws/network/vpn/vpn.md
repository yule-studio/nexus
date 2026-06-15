---
title: "AWS VPN (Site-to-Site / Client)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:26:00+09:00
tags:
  - aws
  - network
  - vpn
  - hybrid
---

# AWS VPN

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | VPN hub |

**[[../network|↑ Network]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**Site-to-Site VPN** (온프레 라우터 ↔ AWS VPC) 와 **Client VPN** (개인 디바이스 ↔ VPC). IPsec 기반. Direct Connect 의 저렴한 대안.

## 2. 종류

| 종류 | 의미 | 언제 |
| --- | --- | --- |
| **Site-to-Site** | 라우터 to AWS VGW / TGW | 데이터센터 ↔ VPC |
| **Client VPN** | OpenVPN 호환 클라이언트 to AWS | 직원 원격 접속 |
| **AWS Verified Access** | (대안) Zero-Trust 응용 접근 — VPN X | 신규 클라이언트 접근 권장 |

## 3. Direct Connect vs VPN

| | Site-to-Site VPN | Direct Connect |
| --- | --- | --- |
| 매체 | 인터넷 (IPsec) | 전용선 |
| 대역폭 | 1.25 Gbps / 터널 | 1G / 10G / 100G |
| Latency 안정성 | 변동 | 안정 |
| 비용 | 시간당 $0.05 + GB | 회선 + 포트 시간 (큼) |
| 도입 시간 | 분 | 주 ~ 개월 |

→ 시작은 **VPN**, 안정 / 대규모는 **DX**. 둘 다 두는 게 표준 (백업).

## 4. 함정

- 1 터널 = 1.25 Gbps — 더 필요하면 multiple tunnels + ECMP
- BGP 라우팅 활용 (정적 routing 보다)
- Verified Access — 신규 zero-trust 패턴 (VPN 없이 응용별 접근)

## 5. 관련

- [[../network|↑ Network]]
- [[../direct-connect/direct-connect]]
- [[../transit-gateway/transit-gateway]]
