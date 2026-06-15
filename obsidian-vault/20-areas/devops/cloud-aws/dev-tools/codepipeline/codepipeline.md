---
title: "AWS CodePipeline"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:32:00+09:00
tags:
  - aws
  - dev-tools
  - cicd
  - pipeline
---

# AWS CodePipeline

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | CodePipeline hub |

**[[../dev-tools|↑ Dev Tools]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

CI/CD 파이프라인 오케스트레이션 — **Source → Build → Test → Deploy** 단계를 그래픽으로 묶는다. GitHub Actions workflow 의 AWS 버전.

## 2. 왜 / 언제

- AWS 안에서 완결된 CI/CD
- 단계 사이 수동 승인 (`manual approval action`)
- 여러 환경 (dev → staging → prod) 순차 배포
- 멀티 region / 멀티 account 배포 자동화

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Pipeline** | 전체 흐름 |
| **Stage** | 단계 (Source / Build / Deploy ...) |
| **Action** | 단계 안 작업 (1개 이상 병렬 가능) |
| **Artifact** | 단계 간 전달되는 파일 (S3 에 자동 저장) |
| **Source providers** | CodeCommit / GitHub (via CodeStar Connection) / S3 / ECR |
| **Variables** | 단계 간 값 전달 |

## 4. 표준 흐름

```
Source (GitHub push)
  ↓
Build (CodeBuild)
  ↓
Test (CodeBuild — 통합 테스트)
  ↓
Deploy to staging (CodeDeploy / ECS / Lambda)
  ↓
Manual approval ←──── Slack 알림 + 사람 승인
  ↓
Deploy to prod (CodeDeploy Blue/Green)
```

## 5. 비용 / 함정

- $1 / active pipeline / 월 (첫 pipeline 30일 무료)
- **GitHub Actions vs CodePipeline** — GHA 가 보통 UX 우수 + 무료. CodePipeline 은 IAM role / VPC / 멀티 account 친화.
- **artifact 만 4MB** 작아 보임 — 큰 파일은 S3 reference
- 멀티 account 배포는 cross-account IAM role 필수
- failure 시 알림 — EventBridge `CodePipeline Pipeline Execution State Change` → SNS

## 6. 관련

- [[../dev-tools|↑ Dev Tools]]
- [[../codebuild/codebuild]]
- [[../codedeploy/codedeploy]]
- [[../../app-integration/eventbridge/eventbridge]]
