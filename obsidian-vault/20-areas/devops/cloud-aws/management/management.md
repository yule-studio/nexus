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

### 1.1 거버넌스 / 컴플라이언스

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Config** | [[config/config\|config]] | 자원 설정 변경 이력 + 규정 준수 룰 자동 평가 (CIS / PCI / HIPAA) |
| **Control Tower** | [[control-tower/control-tower\|control-tower]] | Organizations + Identity Center + Config + CloudTrail 을 landing zone 으로 자동 셋업 |
| **Service Catalog** | [[service-catalog/service-catalog\|service-catalog]] | 관리자 승인 IaC 템플릿(CloudFormation)을 self-service 카탈로그로 제공 |

### 1.2 운영 / 헬스 / 모범사례

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Health Dashboard** | [[health/health\|health]] | AWS 서비스 장애 / 점검 / 보안 알림 통합 대시보드 (Health API + EventBridge) |
| **Trusted Advisor** | [[trusted-advisor/trusted-advisor\|trusted-advisor]] | 비용 / 성능 / 보안 / 한도 / 운영 베스트 프랙티스 자동 체크 (Business+ 플랜) |
| **Well-Architected Tool** | [[well-architected-tool/well-architected-tool\|well-architected-tool]] | Well-Architected 6 pillar 자가 점검 워크북 + HRI 도출 |

### 1.3 자원 / 라이선스

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Resource Access Manager (RAM)** | [[ram/ram\|ram]] | 자원(Subnet / TGW / License 등)을 다른 AWS account 와 공유 (복제 X) |
| **Resource Groups & Tag Editor** | [[resource-groups/resource-groups\|resource-groups]] | 태그 기반 자원 묶기 + 여러 자원 태그 일괄 편집 |
| **License Manager** | [[license-manager/license-manager\|license-manager]] | Windows / SQL Server / Oracle / SAP 라이선스 추적 + BYOL 한도 강제 |

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
