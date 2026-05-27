---
title: "NLB 01-3 — Pricing + Limits"
kind: knowledge
status: current
created_at: 2026-05-23T10:34:00+09:00
tags: [aws, nlb, fundamentals, pricing, nlcu, limits]
home_hub: nlb
---

# NLB Pricing + Limits

> $0.0225/h + NLCU + Cross-zone DT. Hard limits 표.

## 1. 가격 구성

```
NLB 자체:    $0.0225/h × 730 = $16.4/월 (고정)
+ NLCU:      $0.006/NLCU·hour
+ Cross-zone DT: $0.01/GB (ON 시)
+ EIP:       NLB 의 EIP = 무료
```

## 2. NLCU (Network Load Balancer Capacity Unit)

```
1 NLCU = max:
- 800 new TCP connections / sec
- 100K active TCP connections
- 1 GB / hour
- 800 new UDP packets / sec  
- 50 new TLS connections / sec
```

### 시뮬레이션
```
1000 RPS TCP + 50K active + 10 GB/h:
- new TCP: 1000/800 = 1.25 NLCU
- active:  50K/100K = 0.5 NLCU
- traffic: 10/1 = 10 NLCU ← 가장 큼
- TLS:     없음

NLCU = max(...) = 10
Cost = 10 × $0.006 × 730 = $43.8/월
+ Fixed $16
= $60/월
```

## 3. ALB vs NLB 비용 비교

```
같은 100 RPS + 30 GB/h:

ALB LCU: 30 (traffic 1 GB/h dominant)
ALB cost: 30 × $0.008 = $0.24/h × 730 + $16 = $191/월

NLB NLCU: 30 (same metric)
NLB cost: 30 × $0.006 = $0.18/h × 730 + $16 = $148/월

→ 같은 traffic 시 NLB 가 ~25% 싸다 (NLCU price ↓)
```

> ALB 가 routing / WAF / Lambda 의 이점. NLB 는 TCP / static IP / 비용 효율.

## 4. Hard Limits

| | |
| --- | --- |
| NLB / region | 50 (soft, max 1000) |
| Listeners / NLB | 50 |
| Target Groups / NLB | 100 |
| Targets / Target Group | 1000 (instance) / 55K (IP) / 5 (ALB) |
| Subnets / AZ | 1 per AZ |
| Min subnets | 2 (AZ) |
| EIP / NLB | 1 per AZ |
| TLS Cert / Listener | 25 |
| ALPN Policy | 1 per Listener |
| Cross-zone | enable 가능 |
| TCP listener idle timeout | 350s (변경 불가) |
| UDP listener flow timeout | 120s (변경 불가) |

## 5. Service Quotas 증액

```bash
aws service-quotas list-service-quotas --service-code elasticloadbalancing \
  --query "Quotas[?contains(QuotaName, 'Network')]"

aws service-quotas request-service-quota-increase \
  --service-code elasticloadbalancing \
  --quota-code L-69A177A2 \
  --desired-value 100
```

## 6. NLB vs ALB cost decision

```
LCU 의 RuleEvaluations / NewConnection 우세 시 (HTTP API):
  ALB 가 더 정확한 pricing

NLCU 의 ActiveTCP / NewTCP 우세 시 (long-lived TCP):
  NLB 가 더 효율적

같은 GB/h 의 traffic 만 dominant 시:
  NLB 가 25% 싸다
```

## 7. EIP 비용

```
NLB attached EIP:    무료 (NLB 의 일부)
Standalone EIP:      $0.005/h (2024+, 사용 중)
Detached EIP:        $0.005/h
```

## 8. Cross-zone DT

```
Same AZ:     $0
Cross-zone:  $0.01/GB (NLB)
NAT GW DT:   $0.045/GB

→ NLB cross-zone < NAT cost
```

## 9. CloudWatch metric

| Metric | 의미 |
| --- | --- |
| `ActiveFlowCount` | active TCP/UDP |
| `NewFlowCount` | new connection |
| `ProcessedBytes` | 양 |
| `PacketsReceived` | UDP packets |
| `TCP_Client_Reset_Count` | client RST |
| `TCP_Target_Reset_Count` | target RST |
| `HealthyHostCount` / `UnHealthyHostCount` | |
| `ConsumedLCUs` | 실제 NLCU |
| `ClientTLSNegotiationErrorCount` | TLS handshake fail |

### Alarm 표준

```bash
aws cloudwatch put-metric-alarm --alarm-name nlb-unhealthy \
  --metric-name UnHealthyHostCount --namespace AWS/NetworkELB \
  --statistic Maximum --period 60 --evaluation-periods 3 \
  --threshold 1 --comparison-operator GreaterThanOrEqualToThreshold \
  --dimensions Name=LoadBalancer,Value=net/bkurs-nlb/... Name=TargetGroup,Value=targetgroup/...

# TLS handshake fail
aws cloudwatch put-metric-alarm --alarm-name nlb-tls-error \
  --metric-name ClientTLSNegotiationErrorCount --namespace AWS/NetworkELB \
  --statistic Sum --period 60 --evaluation-periods 5 \
  --threshold 10 --comparison-operator GreaterThanThreshold
```

## 10. NLB Access Log (옵션, 2023+)

```bash
aws elbv2 modify-load-balancer-attributes --load-balancer-arn $NLB_ARN \
  --attributes \
    Key=access_logs.s3.enabled,Value=true \
    Key=access_logs.s3.bucket,Value=bkurs-nlb-logs \
    Key=access_logs.s3.prefix,Value=bkurs
```

> NLB Access Log = TLS listener 만 (TCP 는 metadata 적음). TLS 의 connection / source IP / cert 정보.

## 11. Cost Allocation Tag

```bash
aws elbv2 add-tags --resource-arns $NLB_ARN \
  --tags Key=Environment,Value=prod Key=Service,Value=bkurs Key=CostCenter,Value=engineering
```

## 12. 비용 절감

```
1. Cross-zone OFF (per-AZ stickiness) — 가능한 경우
2. Same-AZ task placement
3. ALB 와 비교 — NLB 가 적합한지 검증
4. NLB-as-Target (Chain) — NLB static IP + ALB L7 routing 결합
5. PrivateLink Endpoint Service — Internet 우회
6. EIP NLB attached 만 (standalone 회피)
```

## 13. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Cross-zone OFF + 불균등 | enable + cost 측정 |
| EIP standalone (detach 후 release 안 함) | release |
| TCP listener idle 350s 변경 시도 | 변경 불가 |
| Target 55K (IP) 한도 도달 | sharding |
| NLB Access Log 의 TCP 의미 X | TLS listener 만 |

## 14. 다음

- ▶ [[../02-features/01-tls-and-mtls]]
- ▶ [[../02-features/02-privatelink-endpoint-service]]
- ▶ [[../practice/01-tcp-load-balance]]

## 15. 영상 / 자료

| 자료 |
| --- |
| [NLB Pricing](https://aws.amazon.com/elasticloadbalancing/pricing/) |
| [NLB Quotas](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-limits.html) |

## 16. 참고

- [[../nlb]]
- [[01-listener-and-target-group]]
- [[02-static-ip-and-cross-zone]]
