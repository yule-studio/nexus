---
title: "AWS Compute Optimizer"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:59:00+09:00
tags:
  - aws
  - cost
  - rightsizing
  - compute-optimizer
---

# AWS Compute Optimizer

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Compute Optimizer hub |

**[[../cost|↑ Cost Management]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

EC2 / EBS / Lambda / Auto Scaling / ECS (Fargate) / RDS 의 **사용 패턴 ML 분석 → rightsizing 추천**. "이 m5.4xlarge 는 m5.large 로 충분" 같은 식.

## 2. 왜 / 언제

- 과다 프로비저닝 (over-provisioning) 으로 새는 비용 찾기
- under-provisioning 도 (성능 문제로 사용자 영향)
- EBS gp2 → gp3 마이그레이션 추천
- Lambda 메모리 (= vCPU) 튜닝
- ECS Fargate 의 task 크기 조정

## 3. 추천 대상

| 자원 | 추천 |
| --- | --- |
| **EC2 / ASG** | 인스턴스 타입 변경 / 다운사이즈 |
| **EBS Volume** | gp2 → gp3, IOPS 조정 |
| **Lambda** | 메모리 / 동시성 |
| **ECS Service (Fargate)** | task CPU / 메모리 |
| **RDS** | (Idle / over-provisioned 알림) |

## 4. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Recommendation** | Underprovisioned / Optimized / Overprovisioned / Notoptimized |
| **Saving estimate** | 추천 적용 시 예상 절감 ($/월) |
| **Performance risk** | 권장 변경의 성능 영향 (Very Low ~ High) |
| **Lookback period** | 14일 / 32일 / 93일 (Enhanced 무료 vs 유료) |
| **Enhanced infrastructure metrics** | CloudWatch Agent 메트릭 (메모리 등) 추가 — 더 정확 |
| **External Metrics** | Datadog / Dynatrace 등 외부 데이터 |

## 5. 표준 워크플로우

1. 모든 region 활성화 (콘솔 > Compute Optimizer > Get started)
2. 14일 대기 (데이터 수집)
3. 추천 export → S3 → Athena
4. **performance risk = Very Low + saving > $X** 자동 적용
5. 메모리 메트릭 정확도 위해 **CloudWatch Agent** 설치 권장

## 6. 비용 / 함정

- **무료** (Standard). Enhanced infrastructure metrics 도 무료.
- 메모리 메트릭 없으면 CPU + network 만 — 메모리 bound 워크로드 추천 부정확. CloudWatch Agent 필수.
- 추천 적용은 IaC 로 (CDK / Terraform) — 콘솔 수동은 drift
- ECS task 변경은 무중단 배포로 (배포 자동화 같이)
- 한 번에 다 적용 X — performance risk 낮은 것 / 작은 자원부터

## 7. 관련

- [[../cost|↑ Cost Management]]
- [[../savings-plans/savings-plans]]
- [[../cost-explorer/cost-explorer]]
- [[../../compute/ec2/ec2]]
- [[../../storage/ebs/ebs]]
