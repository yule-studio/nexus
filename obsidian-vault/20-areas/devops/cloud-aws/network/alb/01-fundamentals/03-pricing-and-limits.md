---
title: "ALB 01-3 — Pricing + Limits"
kind: knowledge
status: current
created_at: 2026-05-23T10:00:00+09:00
tags: [aws, alb, fundamentals, pricing, lcu, limits]
home_hub: alb
---

# ALB Pricing + Limits

> $0.0225/h (fixed) + LCU + DT. LCU = max(4 metric). Hard limits 표.

## 1. 가격 구성

```
ALB 자체:  $0.0225/h × 730 = $16.4/월 (고정)
+ LCU:    $0.008/LCU·hour
+ DT:     일반 EC2 DT (cross-AZ / internet)
```

## 2. LCU (Load Balancer Capacity Unit)

```
1 LCU = max 중 가장 큰 값:
- 25 new connections / sec
- 3000 active connections
- 1 GB / hour
- 1000 rule evaluations / sec
```

### 시뮬레이션
```
100 RPS new + 5K active + 30 GB/h:
- new conn: 100/25 = 4 LCU
- active:    5000/3000 = 1.67 LCU
- traffic:   30/1 = 30 LCU ← 가장 큼
- rules:     100×10/1000 = 1 LCU

LCU = max(4, 1.67, 30, 1) = 30
Cost = 30 × $0.008 × 730 = $175/월
+ Fixed $16
= $191/월
```

→ 큰 traffic 의 ALB = LCU 가 dominant.

## 3. Hard Limits

| | |
| --- | --- |
| ALB / region | 50 (soft, max 100) |
| Listeners / ALB | 50 |
| Rules / ALB | 100 (soft) |
| Conditions / Rule | 5 (각 5 values) |
| Target Groups / ALB | 100 |
| Targets / Target Group | 1000 (instance) / 1000 (IP) |
| Subnets / AZ | 1 per AZ |
| Min subnets | 2 (AZ) |
| Max subnets | 1 per AZ |
| Certificate per Listener | 25 |
| Cookie size | 4 KB |
| Cookie name length | 32 chars |
| Slow start time | 0 (off) - 900s |
| Idle timeout | 1 - 4000s (default 60) |

## 4. Service Quotas 증액

```bash
aws service-quotas list-service-quotas --service-code elasticloadbalancing \
  --query 'Quotas[?Adjustable==`true`].[QuotaName,Value]' --output table

aws service-quotas request-service-quota-increase \
  --service-code elasticloadbalancing \
  --quota-code L-53DA6B97 \
  --desired-value 100
```

## 5. Cost 절감

```
1. Cross-zone (ALB 무료) — NLB 와 다름
2. CloudFront 활용 (DT 절감 + cache)
3. ALB Idle Timeout 적절히
4. Rule 통합 (단일 rule + multiple condition)
5. Slow start (warmup) 활용 — health check 의 false negative 회피
6. Multi-app on single ALB (group.name in EKS)
7. Stickiness 의 cookie size 작게
8. Connection draining 짧게 (30s, default 300)
9. Log retention 적정
```

## 6. CloudWatch metric

```
RequestCount               (per ALB / TG)
HTTPCode_Target_2XX/3XX/4XX/5XX_Count
HTTPCode_ELB_4XX/5XX_Count
TargetResponseTime         (latency p50/p90/p99)
HealthyHostCount / UnHealthyHostCount
NewConnectionCount / ActiveConnectionCount / RejectedConnectionCount
ProcessedBytes
ConsumedLCUs
RuleEvaluations
```

### Alarm 표준

```bash
# 5xx rate
aws cloudwatch put-metric-alarm --alarm-name bkurs-alb-5xx \
  --metric-name HTTPCode_ELB_5XX_Count --namespace AWS/ApplicationELB \
  --statistic Sum --period 60 --evaluation-periods 5 \
  --threshold 10 --comparison-operator GreaterThanThreshold \
  --dimensions Name=LoadBalancer,Value=app/bkurs-prod/...

# Target 5xx
aws cloudwatch put-metric-alarm --alarm-name bkurs-target-5xx \
  --metric-name HTTPCode_Target_5XX_Count --namespace AWS/ApplicationELB \
  --statistic Sum --period 60 --evaluation-periods 5 \
  --threshold 10 --comparison-operator GreaterThanThreshold

# Latency p99
aws cloudwatch put-metric-alarm --alarm-name bkurs-latency-p99 \
  --metric-name TargetResponseTime --namespace AWS/ApplicationELB \
  --extended-statistic p99 --period 60 --evaluation-periods 5 \
  --threshold 1 --comparison-operator GreaterThanThreshold

# Unhealthy hosts
aws cloudwatch put-metric-alarm --alarm-name bkurs-unhealthy \
  --metric-name UnHealthyHostCount --namespace AWS/ApplicationELB \
  --statistic Maximum --period 60 --evaluation-periods 3 \
  --threshold 1 --comparison-operator GreaterThanOrEqualToThreshold
```

## 7. 비용 모니터링

```bash
# LCU 사용량
aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB \
  --metric-name ConsumedLCUs \
  --dimensions Name=LoadBalancer,Value=app/bkurs-prod/abc \
  --period 3600 --statistics Sum,Average \
  --start-time $(date -u -v -1d +"%Y-%m-%dT%H:%M:%SZ") \
  --end-time $(date -u +"%Y-%m-%dT%H:%M:%SZ")
```

## 8. Cost Allocation Tag

```bash
aws elbv2 add-tags --resource-arns $ALB_ARN \
  --tags Key=Environment,Value=prod Key=Service,Value=bkurs Key=CostCenter,Value=engineering
```

## 9. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| LCU 큰 traffic 인지 못 함 | metric monitor |
| Idle timeout 너무 짧음 (60s) → WebSocket disconnect | 3600s (WebSocket) |
| Rule 100 한도 도달 | service 분리 |
| TG 1000 target 한도 | sharding |
| Connection draining 300s default | 30s |
| Log retention 무한 | 30-90d |

## 10. 다음

- ▶ [[../02-features/01-https-and-acm]]
- ▶ [[../02-features/02-sticky-sessions-and-health-check]]
- ▶ [[../02-features/03-waf-and-shield]]

## 11. 영상 / 자료

| 자료 |
| --- |
| [ALB Pricing](https://aws.amazon.com/elasticloadbalancing/pricing/) |
| [LCU 계산기](https://aws.amazon.com/blogs/networking-and-content-delivery/understanding-load-balancer-capacity-units/) |

## 12. 참고

- [[../alb]]
- [[01-listener-and-target-group]]
- [[02-routing-rules-and-conditions]]
