---
title: "AWS CodeDeploy"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:31:00+09:00
tags:
  - aws
  - dev-tools
  - cicd
  - deploy
---

# AWS CodeDeploy

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | CodeDeploy hub |

**[[../dev-tools|↑ Dev Tools]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

EC2 / ECS / Lambda 에 **Blue/Green 또는 Canary 배포 자동화** + 자동 rollback. ECS / Lambda 가 가장 자주 쓰임.

## 2. 왜 / 언제

- 새 버전 배포 시 점진 트래픽 전환 (10% → 100%)
- 알람 발생 시 자동 rollback
- pre-traffic / post-traffic Lambda hook (스모크 테스트)
- 무중단 배포 (Blue/Green)

## 3. 배포 방식

| 종류 | 의미 |
| --- | --- |
| **In-place** (EC2/온프레만) | 기존 인스턴스에 직접 덮어쓰기 (간단, 중단 가능) |
| **Blue/Green** | 새 환경에 배포 후 트래픽 스왑 (EC2/ECS/Lambda) |
| **Canary** (Lambda/ECS) | X% → 잠시 후 100% (예: 10% 5분 → 100%) |
| **Linear** | 일정 간격으로 N% 씩 증가 |
| **All-at-Once** | 한 번에 100% |

## 4. ECS Blue/Green 표준

```
ECS Service → ALB target group Blue
                              ↓ CodeDeploy
                       새 task set 생성 → target group Green
                              ↓ 트래픽 10% → 알람 OK → 100%
                              ↓
                       Blue 제거
```

- ALB 의 listener 가 weight 로 분배
- CloudWatch 알람과 연동 — 5xx 증가 시 자동 rollback

## 5. 비용 / 함정

- **무료** (Lambda 배포). EC2 인스턴스당 배포 $0.02 / 배포
- 알람 정의 필수 — 알람 없으면 rollback 안 됨
- IAM role 정확히 — CodeDeploy role + ECS role + Lambda hook role
- AppSpec 파일 (`appspec.yml`) — 배포 hook 정의
- **Blue/Green** 비용 일시 2배 (잠시) — 짧게 끝나도록 짧은 wait time

## 6. 관련

- [[../dev-tools|↑ Dev Tools]]
- [[../codebuild/codebuild]]
- [[../codepipeline/codepipeline]]
- [[../../compute/ecs/ecs]]
- [[../../compute/lambda/lambda]]
