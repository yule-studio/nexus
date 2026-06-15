---
title: "AWS Marketplace"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:31:00+09:00
tags:
  - aws
  - customer-enablement
  - marketplace
---

# AWS Marketplace

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Marketplace hub |

**[[../customer-enablement|↑ Customer Enablement]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

3rd party 소프트웨어 / SaaS / AMI / 컨테이너 / 데이터 / Bedrock 모델 **카탈로그 + AWS 청구서로 통합 결제**. 조달 (procurement) 간소화.

## 2. 왜 / 언제

- 사내 조달 절차 회피 — AWS 청구서 한 장에 SaaS 통합
- Private Marketplace — 사내 승인된 SW 만 노출 (Organizations)
- AWS commitment / EDP 의 일부로 사용 가능
- Datadog / MongoDB / Snowflake 등 SaaS 결제

## 3. 함정

- AWS commitment (EDP) 적용 여부 vendor 별 확인
- 라이선스 형태 (BYOL vs 시간당 vs 약정) 사전 검토
- 일부 listing 의 IAM 권한 매우 광범위 — 검토

## 4. 관련

- [[../customer-enablement|↑ Customer Enablement]]
- [[../../management/license-manager/license-manager]]
- [[../../cost/billing/billing]]
