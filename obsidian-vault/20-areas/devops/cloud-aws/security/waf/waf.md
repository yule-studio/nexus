---
title: "AWS WAF (Web Application Firewall)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:11:00+09:00
tags:
  - aws
  - security
  - waf
  - l7
---

# AWS WAF

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | WAF hub |

**[[../security|↑ Security]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**L7 (HTTP) 방화벽** — SQL injection / XSS / bot / rate limit / 국가 차단 룰을 ALB / CloudFront / API Gateway 앞단에 배치.

## 2. 왜 / 언제

- OWASP Top 10 공격 (SQLi / XSS / RCE) 자동 차단
- 봇 / 스크래퍼 차단 (Bot Control)
- IP / 국가 단위 allowlist / blocklist
- rate limit (`5분에 100건 초과 → 차단`)
- credential stuffing / account takeover 방지

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Web ACL** | 룰 묶음. 1개를 ALB / CloudFront 에 attach |
| **Rule** | 단일 조건 → ALLOW / BLOCK / COUNT / CAPTCHA |
| **Rule Group** | 룰 묶음 (재사용) |
| **AWS Managed Rules** | AWS 가 관리하는 룰셋 (Core / SQLi / KnownBadInputs 등) — **유료** |
| **Marketplace Rules** | F5 / Fortinet / Imperva 등 |
| **Bot Control** | 봇 분류 + 차단 |
| **Captcha / Challenge** | 의심 트래픽에 JS / 캡차 |

## 4. 표준 시작 셋업

```
Web ACL
├── AWS Managed: CommonRuleSet      (SQLi/XSS 기본)
├── AWS Managed: KnownBadInputs
├── AWS Managed: AmazonIpReputation
├── Rate-based rule: 5분 / IP / 2000
└── Default action: ALLOW
```

→ COUNT 모드로 1주일 관찰 → false positive 제거 → BLOCK 전환.

## 5. WAF vs Shield vs Security Group / NACL

| | WAF | Shield | SG / NACL |
| --- | --- | --- | --- |
| 레이어 | L7 (HTTP) | L3-L7 DDoS | L3-L4 (port) |
| 대상 | ALB / CloudFront / API Gateway | 위 + Route 53 / Global Accelerator | EC2 / Subnet |
| 비용 | Web ACL + 룰 + 요청 | Standard 무료 / Advanced $3000/월 | 무료 |

## 6. 비용 / 함정

- Web ACL $5/월, 룰 $1/월, 요청 $0.60 / million
- Managed Rules 추가 $1/월/룰
- **CloudFront 의 WAF 는 global (us-east-1) region**
- **COUNT → BLOCK** 단계적 전환 필수 (정상 사용자 차단 사고)
- Captcha 는 응용 UX 영향 — 의심 트래픽 한정

## 7. 관련

- [[../security|↑ Security]]
- [[../shield/shield]]
- [[../../network/cloudfront/cloudfront]]
- [[../../network/alb/alb]]
