---
title: "AWS Resource Groups & Tag Editor"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:24:00+09:00
tags:
  - aws
  - management
  - tags
  - resource-groups
---

# AWS Resource Groups & Tag Editor

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Resource Groups hub |

**[[../management|↑ Management]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**태그 기반으로 자원을 묶고** 한 화면에서 보기. + Tag Editor 로 여러 자원의 태그 일괄 편집. 비용 / 운영 / 자동화의 기반.

## 2. 왜 / 언제

- 프로젝트 / 환경 / 팀별 자원을 한 view 로
- 비용 분배 (Cost Allocation Tag) 의 사전 작업
- SSM Run Command / Patch Manager 대상 선택
- Backup 정책 적용 대상 자원 묶음
- 누락된 태그 자원 식별

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Tag** | `key=value` (예: `Env=prod`) |
| **Resource Group** | 태그 기반 자원 묶음 (정적 또는 query 기반) |
| **Tag Editor** | 여러 자원의 태그 일괄 add / edit / remove |
| **Tag Policy** (Organizations) | 표준 태그 키 / 값 강제 |
| **AWS Config 룰** (`required-tags`) | 태그 미적용 자원 감지 |

## 4. 표준 태그 정책

```
필수 태그
  Project     = <repo-name>
  Owner       = <team-or-email>
  Env         = prod / staging / dev
  Cost-Center = <부서>
선택 태그
  AppVersion  = v1.2.3
  AutoOff     = 2026-12-31  (스케줄 종료)
```

→ Tag Policy + Config 룰 (`required-tags`) 로 일관성 강제.

## 5. 표준 활용

| 목적 | 사용 |
| --- | --- |
| 비용 분배 | Cost Allocation Tag → [[../../cost/cost-explorer/cost-explorer]] group by tag |
| 일괄 패치 | SSM Patch Manager 대상 = Resource Group |
| 백업 | AWS Backup 의 backup plan 대상 = tag |
| 자동화 | Lambda + EventBridge 가 tag 로 대상 필터 |
| Auto Scaling | tag 기반 정책 |

## 6. 비용 / 함정

- **무료** (Resource Groups / Tag Editor)
- 태그 활성화 = Billing → Cost Allocation Tags 에서 키 활성화 (24h)
- 자동 자원 (예: CloudFormation 이 만든) 의 태그 일관성은 IaC 가 보장
- 태그 너무 많으면 관리 비용 — 5~7 개 표준화

## 7. 관련

- [[../management|↑ Management]]
- [[../organizations/organizations]] — Tag Policy
- [[../config/config]] — required-tags 룰
- [[../../cost/cost-explorer/cost-explorer]]
