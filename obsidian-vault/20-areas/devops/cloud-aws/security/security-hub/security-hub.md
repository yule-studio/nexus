---
title: "AWS Security Hub"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:15:00+09:00
tags:
  - aws
  - security
  - security-hub
  - compliance
---

# AWS Security Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Security Hub hub |

**[[../security|↑ Security]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

여러 보안 서비스 (GuardDuty / Inspector / Macie / IAM Access Analyzer / 3rd party) 의 finding 을 **단일 view + 표준 포맷 (ASFF)** 으로 통합 + 컴플라이언스 점검.

## 2. 왜 / 언제

- 보안 자세 (posture) 한눈에 — score / finding 분포
- 멀티 account 통합 (Organizations 위)
- AWS Foundational Security Best Practices 자동 점검
- PCI / CIS / NIST 표준 컴플라이언스 점검
- 3rd party (Palo Alto / CrowdStrike / Trend Micro 등) 통합

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **ASFF** | AWS Security Finding Format — 표준 JSON |
| **Standards** | 점검 룰 묶음 (AWS FSBP / CIS / PCI DSS / NIST) |
| **Insight** | 사용자 정의 finding 그룹 (예: "이번 주 High") |
| **Control** | standard 안 단위 점검 항목 |
| **Integration** | GuardDuty / Inspector / Macie / IAM Access Analyzer / 3rd party |
| **Automation Rules** | finding 자동 처리 (suppress / 워크플로우 변경) |

## 4. 표준 패턴 — 멀티 account

```
Security account (admin)
  ↑ findings aggregate
  ├── prod account 1
  ├── prod account 2
  └── dev account
```

- Organizations 의 Security account 에 Hub admin 위임
- 모든 account 의 finding 이 한 view 로 보임
- finding 우선순위는 severity + 횟수

## 5. 비용 / 함정

- 비용 — 컴플라이언스 점검 횟수 + finding ingest 횟수 기반
- 자동 활성화 standard 너무 많으면 노이즈 — 처음엔 **AWS FSBP** 만 켜기 권장
- finding 처리 워크플로우 (티켓 / Slack / Jira) 정의 안 하면 무용
- Insight 만들고 자동화 룰로 false positive 묶음 처리

## 6. 관련

- [[../security|↑ Security]]
- [[../guardduty/guardduty]]
- [[../inspector/inspector]]
- [[../../management/config/config]]
- [[../../observability/cloudtrail/cloudtrail]]
