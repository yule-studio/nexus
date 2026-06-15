---
title: "AWS Management & Governance (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T10:34:00+09:00
tags:
  - aws
  - management
  - governance
  - operations
  - hub
---

# AWS Management & Governance (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | 카테고리 hub |

**[[../cloud-aws|↑ AWS]]**

> 운영 / 거버넌스 / 멀티 account 도구. 감사 / 모니터링은 [[../observability/observability|Observability]], 보안 도구는 [[../security/security|Security]] 와 [[../iam/iam|IAM]].

---

## 1. 서비스

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Organizations** | [[organizations]] | 여러 AWS account 를 조직 구조로 + SCP |
| **Systems Manager (SSM)** | [[ssm]] | EC2 운영 (Session Manager / Patch / Parameter Store / Run Command) |
| **Config / Control Tower / Trusted Advisor / Health / Service Catalog / Well-Architected Tool / RAM** | (★★☆ ~ ★☆☆, 별도 예정) | 설정 추적 / landing zone / 헬스체크 / 자원 공유 |

---

## 2. multi-account 표준 패턴

```
Organizations (root)
├── Management account (결제 / Organizations 관리)
├── Security account (CloudTrail / GuardDuty / Security Hub 집중)
├── Log Archive account (모든 log 중앙 저장)
├── Workload accounts
│   ├── prod
│   ├── staging
│   └── dev
└── Sandbox accounts (개발자별)
```

- **SCP** 로 account 단위 차단 (예: 위험 region 금지)
- **IAM Identity Center** ([[../iam/identity-center/identity-center]]) 로 SSO
- **Control Tower** 로 자동 landing zone

---

## 3. EC2 운영 — SSM 가 표준

옛: SSH key + 22 port + Bastion
지금: **Session Manager** (SSH 불필요, IAM 권한 + 감사) — 자세히 [[ssm]]

---

## 4. 관련

- [[../cloud-aws|↑ AWS]]
- [[../iam/iam|↗ IAM]]
- [[../observability/observability|↗ Observability]]
- [[../security/security|↗ Security]]
