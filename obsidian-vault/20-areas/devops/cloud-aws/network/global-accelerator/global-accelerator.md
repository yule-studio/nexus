---
title: "AWS Global Accelerator"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:25:00+09:00
tags:
  - aws
  - network
  - anycast
  - global-accelerator
---

# AWS Global Accelerator

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Global Accelerator hub |

**[[../network|↑ Network]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**Anycast IP 2개 + AWS 백본 사용**으로 글로벌 트래픽 가속. 비-HTTP (TCP / UDP / 게임 / VoIP) 응용의 latency 단축 + 자동 failover.

## 2. 왜 / 언제

- 글로벌 사용자가 region 별 latency 차이 큼
- HTTP 가 아닌 응용 (CloudFront 가 안 됨) — TCP / UDP / 게임
- 정적 IP 2개로 endpoint 고정 (방화벽 화이트리스트)
- region 간 failover 빠르게

## 3. CloudFront vs Global Accelerator

| | CloudFront | Global Accelerator |
| --- | --- | --- |
| 대상 | HTTP / HTTPS | TCP / UDP |
| 캐시 | ✅ | X |
| IP | 동적 | **정적 2개** |
| 비용 | data transfer 위주 | endpoint 시간 + 데이터 |

→ 웹 = CloudFront. 비-웹 / 게임 / VoIP = Global Accelerator.

## 4. 함정

- 시간당 $0.025 + GB 비용 — 항상 켜져 있음
- 작은 region 한정 응용엔 과함

## 5. 관련

- [[../network|↑ Network]]
- [[../cloudfront/cloudfront]]
- [[../route53/route53]]
