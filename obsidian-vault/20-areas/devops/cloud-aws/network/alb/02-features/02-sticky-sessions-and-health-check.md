---
title: "ALB 02-2 — Sticky Sessions + Health Check + Idle Timeout"
kind: knowledge
status: current
created_at: 2026-05-23T10:04:00+09:00
tags: [aws, alb, features, sticky-session, health-check, idle-timeout]
home_hub: alb
---

# ALB Sticky Sessions + Health Check

> Sticky (LB cookie / Application cookie) + Health Check fine-tuning + Idle Timeout (WebSocket).

## 1. Sticky Session 종류

| | Duration-based (LB cookie) | Application-controlled |
| --- | --- | --- |
| Cookie 생성 | ALB 자동 (`AWSALB`) | application 이 만든 cookie |
| 사용 | 일반 stateful app | application 의 정밀 제어 |
| Sticky duration | 1s - 7d | application 결정 |
| 권장 | 단순 stateful | stateless 권장 (Redis 등) |

## 2. LB cookie (Duration-based)

```bash
aws elbv2 modify-target-group-attributes --target-group-arn $TG_ARN \
  --attributes \
    Key=stickiness.enabled,Value=true \
    Key=stickiness.type,Value=lb_cookie \
    Key=stickiness.lb_cookie.duration_seconds,Value=3600
```

→ Client 가 처음 connect 시 ALB 가 `AWSALB` cookie 발급. 같은 cookie 의 client 는 같은 target.

| Pros | Cons |
| --- | --- |
| 간단 | Stateless cluster 추천 (failover 어려움) |
| Stateful session 보존 | Session migration 시 cookie 무효 |

## 3. Application cookie

```bash
aws elbv2 modify-target-group-attributes --target-group-arn $TG_ARN \
  --attributes \
    Key=stickiness.enabled,Value=true \
    Key=stickiness.type,Value=app_cookie \
    Key=stickiness.app_cookie.cookie_name,Value=JSESSIONID \
    Key=stickiness.app_cookie.duration_seconds,Value=3600
```

→ application 의 cookie (JSESSIONID 등) 활용. ALB 가 그 값으로 routing.

## 4. Stateless 권장 (Sticky 없이)

```
Session 을 Redis / DynamoDB / JWT 에 저장:
- Any pod / task 가 처리 가능
- Failover 무중단
- Scale-out 자유
- 권장 ★
```

Sticky 의 trade-off:
- ✓ Stateful migration 회피
- ✗ Load imbalance
- ✗ Failover 시 cookie 무효
- ✗ Scale-out 시 비효율

## 5. Health Check Fine-tuning

```bash
aws elbv2 modify-target-group --target-group-arn $TG_ARN \
  --health-check-protocol HTTP \
  --health-check-port traffic-port \
  --health-check-path /actuator/health/readiness \
  --health-check-interval-seconds 15 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --matcher HttpCode=200
```

| 옵션 | 권장 |
| --- | --- |
| `protocol` | HTTP / HTTPS / TCP |
| `port` | traffic-port (TG 의 port) 또는 명시 |
| `path` | /actuator/health/readiness (Spring Boot) |
| `interval` | 15s (default 30s) |
| `timeout` | 5s |
| `healthy-threshold` | 2 |
| `unhealthy-threshold` | 3 (= 45s 동안 unhealthy 면 deregister) |
| `matcher` | 200-299 (range), 200 (single) |

## 6. Liveness vs Readiness

```
Readiness:
- 새 request 받을 준비 됐는가?
- 응답 가능하면 200
- DB connection / external service check 포함
- ALB / EKS Service 가 routing 결정

Liveness:
- application 자체가 살아있는가?
- 단순 ping 같이
- 죽었으면 (200 아님) restart
```

### Spring Boot
```yaml
management:
  endpoints.web.exposure.include: health,info,prometheus
  endpoint.health.probes.enabled: true
  health.livenessstate.enabled: true
  health.readinessstate.enabled: true
```

→ `/actuator/health/liveness` + `/actuator/health/readiness` 별도.

## 7. Health Check Grace Period

```
ECS Service:    --health-check-grace-period-seconds 120
EKS Pod:        readinessProbe.initialDelaySeconds = 60
ASG:            --health-check-grace-period 120
```

→ Application startup 동안 ALB unhealthy 무시. Spring Boot 의 boot time 120s 권장.

## 8. Slow Start

```bash
aws elbv2 modify-target-group-attributes --target-group-arn $TG_ARN \
  --attributes Key=slow_start.duration_seconds,Value=60
```

→ 새 target 이 healthy 되면 60초 동안 점진적 traffic (0 → 100%).

활용:
- JIT compilation warmup
- Cache warm-up
- Connection pool warming

> Sticky 와 동시 사용 X.

## 9. Deregistration Delay (draining)

```bash
aws elbv2 modify-target-group-attributes --target-group-arn $TG_ARN \
  --attributes Key=deregistration_delay.timeout_seconds,Value=30
```

| 값 | 의미 |
| --- | --- |
| 0 | 즉시 (in-flight request 끊김) |
| 30 | 권장 (web 일반) |
| 60-300 | 긴 request (file upload) |
| 3600 | 매우 긴 connection (default) |

> 너무 짧음 = client error. 너무 김 = slow deploy.

## 10. Idle Timeout (Connection)

```bash
aws elbv2 modify-load-balancer-attributes --load-balancer-arn $ALB_ARN \
  --attributes Key=idle_timeout.timeout_seconds,Value=3600
```

| 값 | Use case |
| --- | --- |
| 60 (default) | 일반 HTTP |
| 3600 (1h) | WebSocket / SSE |
| 4000 (max) | 매우 긴 long-polling |

## 11. Connection Idle / Keep-Alive

```
Client                    ALB                       Target
 ──── connection ─────►    ────── connection ──────►
                          
- Client-side idle: idle_timeout 후 ALB 가 close
- Target-side idle: same
- Keep-Alive 활용 권장 (재사용)
```

## 12. Health Check 의 hosting target type 별

### Instance / IP
```
/actuator/health/readiness
```

### Lambda
```bash
--health-check-enabled
# 또는 disabled (Lambda always available)
```

> Lambda target = health check 거의 의미 없음 (Lambda Service 가 always available).

## 13. UnHealthy host 대응

```bash
# Alarm
aws cloudwatch put-metric-alarm --alarm-name bkurs-unhealthy \
  --metric-name UnHealthyHostCount --namespace AWS/ApplicationELB \
  --statistic Maximum --period 60 --evaluation-periods 3 \
  --threshold 1 --comparison-operator GreaterThanOrEqualToThreshold \
  --dimensions Name=LoadBalancer,Value=app/bkurs-prod/... Name=TargetGroup,Value=targetgroup/...
```

자동 대응 (EKS / ECS):
- ALB 가 unhealthy 표시 → target deregister
- ECS / EKS 가 task / pod 종료 → 새로 생성
- Health Check Grace Period 동안 새 task 대기

## 14. Cross-zone Load Balancing (ALB default ON)

```
ALB Cross-zone:
- Default ON (free)
- Target 의 healthy 와 무관하게 분산

NLB Cross-zone:
- Default OFF (per-AZ stickiness)
- ON 시 $0.01/GB extra
```

## 15. Health Check from ALB IP

ALB 의 health check 출발지 = ALB ENI IP (각 AZ subnet 의 EIP). Target SG 의 inbound:
```
SG_TARGET:
  Inbound: TG_ALB SG → port 8080
  (ALB 의 SG reference)
```

또는 ALB 의 subnet CIDR 명시.

## 16. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Sticky 만으로 stateful 유지 | Redis 권장 |
| Health Check grace period 짧음 → bootup 중 kill | 120+ |
| Deregistration delay 300 default 그대로 → slow deploy | 30 |
| Idle Timeout 60 + WebSocket | 3600 |
| Readiness 와 Liveness 같은 endpoint | 분리 |
| Health Check matcher 200-499 | 200-299 만 |

## 17. 다음

- ▶ [[03-waf-and-shield]]
- ▶ [[../practice/01-alb-multi-target]]
- ▶ [[../practice/02-blue-green-and-waf]]

## 18. 영상 / 자료

| 자료 |
| --- |
| [Sticky Sessions](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/sticky-sessions.html) |
| [Target Group attributes](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-attributes.html) |

## 19. 참고

- [[../alb]]
- [[../01-fundamentals/01-listener-and-target-group]]
