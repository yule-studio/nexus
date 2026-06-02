---
title: "AWS Copilot"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:40:00+09:00
tags:
  - aws
  - containers
  - copilot
  - cli
---

# AWS Copilot

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Copilot hub |

**[[../compute|↑ Compute]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

ECS / Fargate / App Runner 배포 **CLI** — `copilot init` 한 줄로 VPC / ALB / ECR / ECS service / pipeline 자동. CDK 보다 가벼움, App Runner 보다 정밀.

## 2. 왜 / 언제

- CDK / CloudFormation 직접 짜기 싫은 컨테이너 응용
- 응용별 환경 (dev / staging / prod) 빠르게
- pipeline (CodePipeline) 까지 한 번에

## 3. 함정

- AWS 의 권장 베스트 프랙티스 강제 — 커스텀 좁음
- 학습 곡선 vs CDK / Terraform 트레이드오프
- 신규는 **AWS Copilot v2** (2024+, CDK 기반)

## 4. 관련

- [[../compute|↑ Compute]]
- [[../ecs/ecs]]
- [[../app-runner/app-runner]]
- [[../../dev-tools/cdk/cdk]]
