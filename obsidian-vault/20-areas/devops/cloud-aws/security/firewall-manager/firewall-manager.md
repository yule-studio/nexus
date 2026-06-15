---
title: "AWS Firewall Manager"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:43:00+09:00
tags:
  - aws
  - security
  - firewall-manager
---

# AWS Firewall Manager

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Firewall Manager hub |

**[[../security|↑ Security]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

Organizations 단위로 **WAF / Shield / Security Group / Network Firewall / Route 53 Resolver DNS Firewall** 룰 일괄 관리. 멀티 account 보안 정책의 중앙 집중.

## 2. 왜 / 언제

- 모든 account 의 ALB / CloudFront 에 같은 WAF 룰 강제
- 표준 SG 룰 강제 (예: "0.0.0.0/0 → 22 차단")
- 신규 자원에도 자동 적용

## 3. 함정

- $100/policy/월 + 룰별 비용 — 큰 조직에 적합
- 작은 조직 = 각 account 에서 직접 WAF / SG 관리
- Organizations + Config + Security Hub 와 한 묶음 운영

## 4. 관련

- [[../security|↑ Security]]
- [[../waf/waf]] / [[../shield/shield]]
- [[../security-hub/security-hub]]
- [[../../management/organizations/organizations]]
