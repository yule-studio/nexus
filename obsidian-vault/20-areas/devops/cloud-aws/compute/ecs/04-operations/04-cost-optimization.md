---
title: "ECS 04-4 — Cost Optimization (Spot / Savings Plan / Right-sizing)"
kind: knowledge
status: current
created_at: 2026-05-22T14:06:00+09:00
tags: [aws, ecs, operations, cost]
home_hub: ecs
---

# ECS Cost Optimization

> 50~70% 절감. 핵심 5: Right-sizing / Graviton (ARM64) / Fargate Spot / Compute Savings Plan / Cluster scheduled shutdown.

## 1. 비용 구조

| 구성 | 비용 |
| --- | --- |
| Fargate | vCPU/RAM 시간 (1vCPU+2GB ≈ $36/월) |
| EC2 launch | EC2 시간 + container 무료 |
| EBS / NAT / ALB / DataTransfer | 각각 |
| CloudWatch Logs | ingestion + storage |
| Container Insights | $1.50/cluster/일 |

## 2. Strategy 1 — Right-sizing

Container Insights 로 측정:
```
CPU avg 25% + Mem 70% → vCPU 줄임
```

도구: AWS Compute Optimizer.

## 3. Strategy 2 — Graviton (ARM64)

```json
{ "runtimePlatform": { "cpuArchitecture": "ARM64" } }
```

| | X86 | ARM64 |
| --- | --- | --- |
| $/h | $0.05 | $0.04 (-20%) |

multi-arch build → [[../../../../docker/practice/06-multi-platform-build]]

## 4. Strategy 3 — Fargate Spot

```json
[
  { "capacityProvider": "FARGATE",      "weight": 1, "base": 3 },
  { "capacityProvider": "FARGATE_SPOT", "weight": 4 }
]
```

| | Fargate | Spot |
| --- | --- | --- |
| $/h | $0.05 | $0.015 (~70% 할인) |
| interruption | 없음 | 2분 warning |

graceful shutdown 필수:
```json
{ "stopTimeout": 120 }
```

## 5. Strategy 4 — Compute Savings Plan

| 약정 | 할인 |
| --- | --- |
| 1년 no upfront | ~17% |
| 1년 all upfront | ~40% |
| 3년 all upfront | ~52% |

> Compute Savings Plan 이 Fargate / EC2 / Lambda 모두 커버. baseline 60~80% 만 약정.

## 6. Strategy 5 — Scheduled shutdown (dev/staging)

```bash
aws application-autoscaling put-scheduled-action \
  --service-namespace ecs --resource-id service/dev/api \
  --scalable-dimension ecs:service:DesiredCount \
  --scheduled-action-name night-shutdown \
  --schedule "cron(0 22 ? * MON-FRI *)" \
  --scalable-target-action MinCapacity=0,MaxCapacity=0
```

→ 야간+주말 ~67% 절감.

## 7. EC2 launch 추가 절감

### Spot ASG
```json
{
  "InstancesDistribution": {
    "OnDemandPercentageAboveBaseCapacity": 20,
    "SpotInstancePools": 4
  }
}
```

### Right-sized instance type
- 작은 task: t3.small / medium
- bin-packing: m6g.large (Graviton)

## 8. NAT Gateway 비용

NAT: $32/월 + $0.05/GB. **VPC Endpoint** 사용:
| Service | Type | 비용 |
| --- | --- | --- |
| S3 / DynamoDB | Gateway | 무료 |
| ECR / SSM / Logs / Secrets Manager | Interface | $7/월/AZ |

## 9. CloudWatch Logs

- Retention 7~30일
- Log level INFO (DEBUG 비활성)
- firelens + Loki 외부 분산
- Sampling (1%)

## 10. Cost Explorer + Tag

권장 tag:
- `Environment` / `Service` / `Team`

→ service 별 비용 visibility.

## 11. 권장 — bkurs-be

```
Fargate ARM64 1 vCPU / 2GB                      → $29/월/task
+ Capacity Provider FARGATE base + SPOT 4      → 약 50% ↓
+ Compute Savings Plan 1y partial               → 25% ↓
+ dev/staging 야간 shutdown                     → dev 67% ↓
+ VPC Endpoint (ECR/SSM/Logs/Secrets)           → NAT ↓
+ Log retention 30 days                         → Logs 80% ↓
```

월 비용 추정: prod 3 task + dev 1 task ≈ **$150/월** (vs 최적화 안 한 ~$300).

## 12. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Over-provisioned | Right-sizing |
| ARM64 image 안 빌드 | multi-arch |
| Spot interruption | base on-demand 보장 |
| 24h dev | scheduled shutdown |
| Log 무제한 | 30일 retention |
| NAT 비용 모름 | VPC Endpoint |

## 13. 다음

- ▶ [[01-ecs-exec]]
- ▶ [[../05-security/01-iam-best-practices]]
- ▶ [[../practice/02-ec2-with-asg]]
- ▶ [[../../../../10-projects/bkurs-be/handover/operations/env-deployment#8]]

## 14. 영상 / 자료

| 자료 |
| --- |
| [ECS Cost Optimization — re:Invent](https://www.youtube.com/watch?v=8q3wQFGqI50) |
| [Graviton — AWS](https://aws.amazon.com/ec2/graviton/) |
| [Compute Optimizer](https://aws.amazon.com/compute-optimizer/) |

## 15. 참고

- [[../ecs]]
- [[../01-fundamentals/03-fargate-deep]]
- [[../03-service/03-auto-scaling]]
