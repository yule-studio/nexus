---
title: "AWS Shield (DDoS Protection)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:12:00+09:00
tags:
  - aws
  - security
  - ddos
  - shield
---

# AWS Shield

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Shield hub |

**[[../security|↑ Security]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

DDoS 공격 방어. **Standard 무료 (자동)**, **Advanced 유료** (월 $3000~, 24/7 DRT 지원 + 비용 보호).

## 2. 왜 / 언제

- L3 / L4 (SYN flood / UDP reflection) 자동 완화 — **Standard, 무료**
- L7 (HTTP flood) 도 완화 + 비용 protection — **Advanced**
- 대규모 공격으로 인한 트래픽 청구를 AWS 가 면제 — **Advanced**
- DRT (DDoS Response Team) 직접 지원 받기

## 3. 두 단계

| | Shield Standard | Shield Advanced |
| --- | --- | --- |
| 비용 | **무료, 자동** | $3,000 / 월 (1년 약정) + 데이터 비용 |
| 보호 | L3/L4 자동 | L3/L4/L7 + WAF 포함 |
| 비용 protection | X | **공격 트래픽 청구 면제** |
| DRT | X | 24/7 응답팀 |
| 보호 대상 | 모든 AWS | CloudFront / Route 53 / GA / ALB / EIP |

## 4. 표준 패턴

- **Standard 만으로 대부분 충분** (특히 CloudFront + Route 53 앞단)
- 대규모 서비스 / 금융 / 게임 / 정치 = **Advanced + WAF**
- Advanced 활성화 시 **WAF 무료 포함** (Web ACL / Managed Rules)

## 5. 비용 / 함정

- Advanced $3000/월 = **연 $36,000** 약정. 작은 서비스에는 과한 비용.
- 비용 protection 의 의미 — DDoS 로 인한 비정상 트래픽 비용 (예: $50,000 추가 청구) 을 AWS 가 환불 / 면제.
- DDoS 대비 = **CloudFront 앞단 + Route 53 + Auto Scaling + Shield Standard** 가 최소.

## 6. 관련

- [[../security|↑ Security]]
- [[../waf/waf]] — Advanced 와 함께
- [[../../network/cloudfront/cloudfront]]
- [[../../network/route53/route53]]
