---
title: "AWS Elastic Beanstalk"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:10:00+09:00
tags:
  - aws
  - compute
  - paas
---

# AWS Elastic Beanstalk

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Elastic Beanstalk hub |

**[[../compute|↑ Compute]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

응용 코드 (zip / war / docker) 업로드만으로 **EC2 + ALB + Auto Scaling + RDS 자동 프로비저닝**. AWS 의 1세대 PaaS. 신규 응용은 ECS / App Runner 가 보통 더 나음.

## 2. 왜 / 언제

- 인프라 직접 짜기 싫은 단순 Java / Python / Node / Ruby / .NET 응용
- 옛 모놀리스 응용을 빠르게 EC2 로
- 자동 CloudWatch + 로그 통합

## 3. 핵심 / 함정

| 개념 | 의미 |
| --- | --- |
| **Application / Environment** | 응용 / 환경 (web / worker) |
| **Platform** | 언어별 사전 빌드 AMI (Amazon Linux 2023 기반) |
| **.ebextensions** | 커스텀 설정 (config 파일) |
| **함정** | platform 버전 업글 잦음 (수동 마이그) — 점차 deprecated 추세 |
| **신규 권장** | App Runner ([[../app-runner/app-runner]]) 또는 ECS Fargate |

## 4. 관련

- [[../compute|↑ Compute]]
- [[../app-runner/app-runner]]
- [[../ecs/ecs]]
