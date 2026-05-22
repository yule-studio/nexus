---
title: "ECS 03-5 — Load Balancer Integration (ALB / NLB target group)"
kind: knowledge
status: current
created_at: 2026-05-22T13:58:00+09:00
tags: [aws, ecs, service, alb, nlb]
home_hub: ecs
---

# ECS Load Balancer Integration

> ECS task → ALB / NLB target group 자동 등록. awsvpc + IP target 표준.

## 1. LB 선택

| 종류 | Layer | 용도 |
| --- | --- | --- |
| **ALB** | 7 (HTTP/S) | HTTP API / WebSocket / gRPC / path routing |
| **NLB** | 4 (TCP/UDP) | TCP 직접 / static IP / 초고성능 |
| **GWLB** | 3 | security appliance |
| **Service Connect** | 7 | internal east-west |

## 2. Target Group — 2 type

| Type | 의미 |
| --- | --- |
| **IP** | task IP 등록 (awsvpc) — 권장 |
| **instance** | EC2 instance ID (bridge — legacy) |

```bash
aws elbv2 create-target-group \
  --name bkurs-api-tg --protocol HTTP --port 8080 \
  --target-type ip --vpc-id vpc-xxx \
  --health-check-path /actuator/health \
  --health-check-interval-seconds 15 \
  --healthy-threshold-count 2 --unhealthy-threshold-count 3 \
  --matcher HttpCode=200
```

## 3. ECS Service 연동

```bash
aws ecs create-service \
  --service-name api \
  --load-balancers "targetGroupArn=arn:...:target-group/bkurs-api-tg/abc,containerName=api,containerPort=8080" \
  --health-check-grace-period-seconds 120
```

→ ECS 가 자동 register / deregister.

## 4. ALB Listener / Rule

```bash
aws elbv2 create-listener \
  --load-balancer-arn arn:... --protocol HTTPS --port 443 \
  --certificates CertificateArn=arn:acm:... \
  --default-actions Type=forward,TargetGroupArn=arn:...

# Path 기반
aws elbv2 create-rule --listener-arn arn:... --priority 10 \
  --conditions Field=path-pattern,Values=/api/v1/auth/* \
  --actions Type=forward,TargetGroupArn=arn:...:tg/auth/...
```

## 5. Multi-Target Group per Service

```json
"loadBalancers": [
  { "targetGroupArn": "arn:...:tg/api-http/...",  "containerName": "api",   "containerPort": 8080 },
  { "targetGroupArn": "arn:...:tg/api-grpc/...", "containerName": "api",   "containerPort": 9090 },
  { "targetGroupArn": "arn:...:tg/admin/...",    "containerName": "admin", "containerPort": 8081 }
]
```

max 5 LB / service.

## 6. health check grace period

```bash
--health-check-grace-period-seconds 120
```

→ task 시작 후 120초 ALB unhealthy 무시 (Spring Boot 시작 시간).

## 7. Deregistration delay (draining)

```bash
--target-group-attributes "deregistration_delay.timeout_seconds=30"
```

| 값 | 의미 |
| --- | --- |
| 0 | 즉시 |
| 30~60 | 일반 web |
| 300 | 긴 요청 |

## 8. Sticky Session

```bash
aws elbv2 modify-target-group-attributes \
  --attributes Key=stickiness.enabled,Value=true \
              Key=stickiness.type,Value=lb_cookie \
              Key=stickiness.lb_cookie.duration_seconds,Value=3600
```

> 일반 권장 X — stateless API 표준.

## 9. TLS

ALB:
- ACM Certificate 자동
- mTLS (2023+) 지원

NLB:
- TLS listener
- pass-through 가능

## 10. Health Check Path

```bash
--health-check-path /actuator/health/readiness
```

Liveness vs Readiness 분리 권장. 자세히 → [[../02-task-definition/08-healthcheck]] §6.

## 11. NLB 활용

```bash
aws elbv2 create-listener \
  --load-balancer-arn arn:nlb:... --protocol TCP --port 5432 \
  --default-actions Type=forward,TargetGroupArn=arn:tg-postgres:...
```

활용: Postgres / Redis / Custom protocol / static IP / gRPC.

## 12. Cross-AZ

```bash
--network-configuration "awsvpcConfiguration={subnets=[subnet-az-a,subnet-az-b,subnet-az-c],securityGroups=[sg-task]}"
```

ECS scheduler 가 AZ balance + spread placement.

## 13. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Target Group `instance` + awsvpc | `ip` 로 변경 |
| target unhealthy 시 task kill | grace period 120+ |
| deregistration delay 길음 | 30s |
| HTTPS 인증서 없음 | ACM + DNS validate |
| HTTP/2 / gRPC 안 됨 | protocol_version HTTP2 / GRPC |
| WebSocket disconnect | idle timeout 3600s |

## 14. CDK

```typescript
const lb = new elbv2.ApplicationLoadBalancer(this, 'LB', { vpc, internetFacing: true });
const listener = lb.addListener('HTTPS', { port: 443, certificates: [cert] });
const tg = listener.addTargets('Api', {
  port: 8080,
  protocol: elbv2.ApplicationProtocol.HTTP,
  targetType: elbv2.TargetType.IP,
  healthCheck: { path: '/actuator/health/readiness', healthyThresholdCount: 2 },
});
service.attachToApplicationTargetGroup(tg);
```

## 15. 다음

- ▶ [[../04-operations/01-ecs-exec]]
- ▶ [[../04-operations/02-container-insights]]
- ▶ [[../../network/alb/alb]] — ALB 일반

## 16. 영상 / 자료

| 자료 |
| --- |
| [ALB + ECS — AWS](https://www.youtube.com/watch?v=qHnFkU2LXcc) |
| [공식 ECS LB](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-load-balancing.html) |

## 17. 참고

- [[../ecs]]
- [[../02-task-definition/03-network-mode]]
- [[../02-task-definition/08-healthcheck]]
