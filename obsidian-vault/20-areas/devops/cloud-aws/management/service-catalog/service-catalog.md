---
title: "AWS Service Catalog"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:30:00+09:00
tags:
  - aws
  - management
  - governance
  - service-catalog
---

# AWS Service Catalog

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Service Catalog hub |

**[[../management|↑ Management]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**관리자가 승인한 IaC 템플릿 (CloudFormation)** 을 사용자에게 self-service 카탈로그로. 표준 자원만 빠르게 띄울 수 있게 가드레일.

## 2. 왜 / 언제

- 개발자가 표준 EC2 / RDS / VPC 를 직접 만들지 않게 → 표준 템플릿만 클릭
- Control Tower 의 Account Factory 도 Service Catalog 위에서
- 라이센스 / 컴플라이언스 제어
- TagOptions 로 비용 분배 자동

## 3. 함정

- 작은 팀에는 과한 거버넌스 — 큰 조직 / 규제 도메인용
- CloudFormation 의 한계가 곧 Service Catalog 의 한계

## 4. 관련

- [[../management|↑ Management]]
- [[../control-tower/control-tower]]
- [[../../dev-tools/cloudformation/cloudformation]]
