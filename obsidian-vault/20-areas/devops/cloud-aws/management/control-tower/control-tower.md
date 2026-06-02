---
title: "AWS Control Tower"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:21:00+09:00
tags:
  - aws
  - management
  - landing-zone
  - control-tower
---

# AWS Control Tower

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Control Tower hub |

**[[../management|↑ Management]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

Organizations + IAM Identity Center + Config + CloudTrail + 로그 집중을 **landing zone** 으로 한 번에 자동 셋업. 신규 account 도 표준 가드레일 자동 적용.

## 2. 왜 / 언제

- 신규 AWS 조직 시작 — 멀티 account 표준 셋업
- 매번 새 account 셋업의 반복 작업을 자동화
- 보안 / 감사 / 로그 / SSO 베스트 프랙티스 강제
- Account Factory 로 self-service account 발급

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Landing Zone** | 멀티 account + Org + 로그 + 보안 표준 묶음 |
| **Guardrail** | preventive (SCP) / detective (Config Rule) 점검 |
| **Account Factory** | Service Catalog 기반 account 자동 발급 |
| **Customizations (CfCT)** | landing zone 위 자체 추가 |
| **Mandatory / Optional Guardrails** | 필수 vs 선택 룰 |

## 4. 자동 생성되는 것

- **Management account** 외 별도:
  - Log Archive account (CloudTrail / Config 중앙 저장)
  - Audit account (Security Hub / GuardDuty)
- 표준 OU 구조 (Security / Sandbox)
- IAM Identity Center 활성화 + 기본 Permission Set
- 모든 account 의 CloudTrail / Config 자동 활성화
- 기본 SCP (root 사용 차단 / region 차단 등)

## 5. 비용 / 함정

- Control Tower 자체 무료, 다만 깔린 서비스 (Config / CloudTrail / S3) 비용 발생
- **한 번 셋업 후 수동 변경 X** — Control Tower 가 reset 함. customization 은 CfCT 로.
- 기존 Organizations 위에 Control Tower 얹기는 가능하지만 제약 있음 — 신규가 깔끔.
- ap-northeast-2 (Seoul) home region 지원 — 옛날엔 us 전용이었음.

## 6. 관련

- [[../management|↑ Management]]
- [[../organizations/organizations]]
- [[../config/config]]
- [[../../iam/identity-center/identity-center]]
