---
title: "Amazon CodeCatalyst"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:34:00+09:00
tags:
  - aws
  - dev-tools
  - sdlc
  - codecatalyst
---

# Amazon CodeCatalyst

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | CodeCatalyst hub |

**[[../dev-tools|↑ Dev Tools]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

AWS 통합 SDLC 포털 — **이슈 / 리포 / 워크플로우 / 환경 (dev environment) / Blueprints**. GitHub + Jira + GH Actions + Codespaces 를 한 데. AWS Builder ID 로 로그인.

## 2. 왜 / 언제

- AWS 친화적 개발자 포털
- dev environment (Cloud9 후신, EC2 dev box)
- Blueprint 로 표준 응용 (React + Lambda + DynamoDB) 1 클릭 생성
- 통합 CI/CD 워크플로우

## 3. 함정

- GitHub 생태계가 훨씬 큼 — 보통 GitHub + Actions 가 디폴트
- AWS 안 lock-in
- 2026 시점에서 사용 빈도 크게 늘지 않음 — 사용 전 도입 비용 / lock-in 검토

## 4. 관련

- [[../dev-tools|↑ Dev Tools]]
- [[../q-developer/q-developer]]
- [[../codepipeline/codepipeline]]
