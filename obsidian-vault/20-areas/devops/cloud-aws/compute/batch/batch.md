---
title: "AWS Batch"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:11:00+09:00
tags:
  - aws
  - compute
  - batch
  - hpc
---

# AWS Batch

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Batch hub |

**[[../compute|↑ Compute]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**대량 배치 작업 (수천 ~ 수만 job)** 의 큐 + 자동 컴퓨팅 확장. ECS / EKS / Fargate 위에서 실행. HPC / ML 학습 / 야간 ETL 의 자원 매니저.

## 2. 왜 / 언제

- 야간 / 주기적 대규모 데이터 처리
- ML 학습 / 추론 batch
- 영상 / 게놈 / HPC 시뮬레이션
- 큐에 쌓인 작업을 우선순위로 처리 + Spot 활용

## 3. 핵심 / 함정

| 개념 | 의미 |
| --- | --- |
| **Job Queue** | 작업 큐 (우선순위) |
| **Compute Environment** | EC2 / Fargate / Spot — 자동 확장 |
| **Job Definition** | 컨테이너 이미지 + 자원 + retry |
| **Array Job** | 1개 정의 → N 인스턴스 (병렬 인덱스) |
| **함정** | Spot 인터럽션 대비 checkpoint 필요. cold start 길 수도 |

## 4. 관련

- [[../compute|↑ Compute]]
- [[../ecs/ecs]]
- [[../../app-integration/step-functions/step-functions]] — 복잡한 의존성
