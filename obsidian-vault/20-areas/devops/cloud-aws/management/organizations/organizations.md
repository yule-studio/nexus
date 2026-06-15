---
title: "AWS Organizations"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T10:45:00+09:00
tags:
  - aws
  - management
  - governance
  - multi-account
---

# AWS Organizations

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Organizations hub |

**[[../management|↑ Management]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

여러 AWS account 를 **조직 (tree) 구조** 로 관리 + **SCP** (Service Control Policy) 로 account 단위 차단 + 통합 결제.

---

## 2. 왜 / 언제

- 환경 분리 (prod / staging / dev / sandbox) — VPC 가 아니라 **account** 단위로
- 폭발 반경 (blast radius) 격리 — prod 사고가 dev 에 안 번짐
- 비용 / 보안 / 로그 / 결제 분리
- 부서별 / 팀별 자율성 + 거버넌스 표준화
- SCP 로 위험 region / 위험 action 강제 차단

---

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Management account** | 최상위 (결제 / Organizations 관리) — 워크로드 두지 않음 |
| **OU (Organizational Unit)** | account 묶음 폴더 |
| **Member account** | 일반 워크로드 account |
| **SCP** | OU / account 에 적용되는 **권한 상한** (deny 만 의미 있음) |
| **Consolidated billing** | 모든 account 의 청구를 management 가 통합 |
| **Service-linked features** | CloudTrail / Config / GuardDuty / Security Hub 등이 Org 단위로 활성화 |

---

## 4. 표준 OU 구조

```
Root
├── Security OU
│   ├── Log Archive account     (CloudTrail / Config 중앙 log)
│   └── Audit account           (GuardDuty / Security Hub)
├── Infrastructure OU
│   └── Shared services account (DNS / TGW / 공통 ECR)
├── Workloads OU
│   ├── Prod OU
│   │   ├── prod-app-a
│   │   └── prod-app-b
│   └── Non-prod OU
│       ├── staging
│       └── dev
└── Sandbox OU
    └── alice-sandbox, bob-sandbox ...
```

→ **Control Tower** 가 이 구조 자동 생성.

---

## 5. SCP 예시 — region 제한

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyNonApprovedRegions",
    "Effect": "Deny",
    "Action": "*",
    "Resource": "*",
    "Condition": {
      "StringNotEquals": {
        "aws:RequestedRegion": ["ap-northeast-2", "us-east-1"]
      },
      "ForAllValues:StringNotEquals": {
        "aws:PrincipalArn": ["arn:aws:iam::*:role/OrganizationAccountAccessRole"]
      }
    }
  }]
}
```

→ Seoul / N.Virginia 외 region 으로 자원 생성 차단.

SCP 흔한 쓰임:
- 위험 region 차단
- root user action 차단
- CloudTrail / Config 비활성화 차단
- 특정 service 차단 (예: 신입 OU 에서 SageMaker 금지)

---

## 6. SCP 평가 순서 (IAM 과 함께)

```
SCP (Deny) → 사용자 IAM (Allow) → 자원 정책 → Permission Boundary
```

→ 어느 하나라도 Deny → 차단. SCP 가 **Allow** 여도 IAM 이 Deny 면 차단 (SCP 는 상한일 뿐).

---

## 7. 함정 / 비용

- **무료** (Organizations 자체). 다만 service 별 비용 (CloudTrail / GuardDuty 등) 은 발생.
- **Management account 에 워크로드 두지 X** — 사고 시 전체 위험.
- **SCP 너무 광범위** — `Deny *` 같은 실수는 자기 자신도 차단. 단계적 rollout.
- **신규 account 자동 SCP 미적용** — Control Tower / SCM 으로 자동화.
- **EC2 / RDS quota 는 account 단위** — 큰 워크로드는 별도 account.

---

## 8. 관련

- [[../management|↑ Management]]
- [[../../iam/iam]]
- [[../../iam/identity-center/identity-center]] — Org 위에서만 동작
- [[../../observability/cloudtrail/cloudtrail]] — Org trail
