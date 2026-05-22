---
title: "ALB 01-1 — Listener + Target Group + Target Type"
kind: knowledge
status: current
created_at: 2026-05-22T21:48:00+09:00
tags: [aws, alb, fundamentals, listener, target-group, target-type]
home_hub: alb
---

# ALB Listener + Target Group

> Listener (80/443) + Target Group (instance / ip / lambda / alb) + Rule + Health Check.

## 1. ALB 생성

```bash
ALB_ARN=$(aws elbv2 create-load-balancer --name bkurs-prod \
  --type application \
  --scheme internet-facing \
  --subnets $PUB_SUBNET_A $PUB_SUBNET_B $PUB_SUBNET_C \
  --security-groups $SG_ALB \
  --ip-address-type ipv4 \
  --tags Key=Environment,Value=prod \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)
```

| 옵션 | 의미 |
| --- | --- |
| `--type` | application / network / gateway |
| `--scheme` | internet-facing / internal |
| `--subnets` | 2+ AZ 필수 |
| `--ip-address-type` | ipv4 / dualstack (IPv6) |

## 2. Listener — HTTP / HTTPS

```bash
# HTTP 80 (redirect to HTTPS)
aws elbv2 create-listener --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions Type=redirect,RedirectConfig='{
    Protocol=HTTPS,
    Port=443,
    StatusCode=HTTP_301
  }'

# HTTPS 443
LISTENER_ARN=$(aws elbv2 create-listener --load-balancer-arn $ALB_ARN \
  --protocol HTTPS --port 443 \
  --certificates CertificateArn=$ACM_CERT \
  --ssl-policy ELBSecurityPolicy-TLS13-1-2-2021-06 \
  --default-actions Type=forward,TargetGroupArn=$TG_DEFAULT_ARN \
  --query 'Listeners[0].ListenerArn' --output text)
```

## 3. Target Group 4 종

| Type | Target |
| --- | --- |
| **Instance** | EC2 instance (port mapping) |
| **IP** | private IP (Fargate / EKS awsvpc) ★ |
| **Lambda** | Lambda function |
| **ALB** | 다른 ALB (NLB → ALB chain) |

### Instance type
```bash
aws elbv2 create-target-group --name tg-api \
  --protocol HTTP --port 8080 \
  --vpc-id $VPC_ID --target-type instance \
  --health-check-path /actuator/health \
  --health-check-interval-seconds 15 \
  --healthy-threshold-count 2 --unhealthy-threshold-count 3
```

### IP type ★ (Fargate / EKS)
```bash
aws elbv2 create-target-group --name tg-api-ip \
  --protocol HTTP --port 8080 \
  --vpc-id $VPC_ID --target-type ip \
  --health-check-path /actuator/health/readiness \
  --health-check-interval-seconds 15 \
  --healthy-threshold-count 2 \
  --target-group-attributes Key=deregistration_delay.timeout_seconds,Value=30
```

### Lambda type
```bash
aws elbv2 create-target-group --name tg-fn \
  --target-type lambda \
  --health-check-enabled

aws elbv2 register-targets --target-group-arn $TG_LAMBDA \
  --targets "Id=arn:aws:lambda:...:function:bkurs-fn"

aws lambda add-permission --function-name bkurs-fn \
  --statement-id alb --action lambda:InvokeFunction \
  --principal elasticloadbalancing.amazonaws.com \
  --source-arn $TG_LAMBDA
```

## 4. Target Group attributes

```bash
aws elbv2 modify-target-group-attributes --target-group-arn $TG_ARN \
  --attributes \
    Key=deregistration_delay.timeout_seconds,Value=30 \
    Key=stickiness.enabled,Value=false \
    Key=load_balancing.algorithm.type,Value=round_robin \
    Key=target_group_health.dns_failover.minimum_healthy_targets.count,Value=1 \
    Key=target_group_health.unhealthy_state_routing.minimum_healthy_targets.percentage,Value=50
```

| Attribute | 의미 |
| --- | --- |
| `deregistration_delay.timeout_seconds` | drain (default 300) |
| `load_balancing.algorithm.type` | round_robin / least_outstanding_requests / weighted_random |
| `stickiness.enabled` | sticky session |
| `target_group_health.dns_failover` | unhealthy target 비율 시 ALB DNS failover |
| `slow_start.duration_seconds` | new target 의 warm-up (gradual traffic) |

## 5. Register Targets

```bash
# Instance
aws elbv2 register-targets --target-group-arn $TG_ARN \
  --targets Id=i-xxx,Port=8080 Id=i-yyy,Port=8080

# IP
aws elbv2 register-targets --target-group-arn $TG_IP \
  --targets Id=10.0.1.50,Port=8080 Id=10.0.2.50,Port=8080

# Lambda (1개만, 자동)
aws elbv2 register-targets --target-group-arn $TG_LAMBDA \
  --targets Id=arn:aws:lambda:...:function:bkurs-fn
```

## 6. Health Check

```bash
aws elbv2 modify-target-group --target-group-arn $TG_ARN \
  --health-check-protocol HTTP \
  --health-check-port 8080 \
  --health-check-path /actuator/health/readiness \
  --health-check-interval-seconds 15 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --matcher HttpCode=200-299
```

| 옵션 | 권장 |
| --- | --- |
| `protocol` | HTTP (gRPC: HTTP, HTTPS for end-to-end TLS) |
| `port` | 자동 (TG port) or `traffic-port` |
| `path` | `/actuator/health/readiness` (Spring Boot) |
| `interval-seconds` | 15-30 |
| `timeout-seconds` | 5-10 |
| `healthy-threshold` | 2 |
| `unhealthy-threshold` | 3 |
| `matcher` | 200-299 |

## 7. Health Check Grace Period (ECS / EKS)

```bash
# ECS
aws ecs create-service --health-check-grace-period-seconds 120 ...

# EKS
spec:
  containers:
    - readinessProbe:
        initialDelaySeconds: 60   # boot 시간
```

> Spring Boot 의 application startup time 고려 (120s 권장).

## 8. ECS Task / EKS Pod 의 자동 registration

```yaml
# ECS Service
loadBalancers:
  - targetGroupArn: arn:...:targetgroup/tg-api-ip/...
    containerName: api
    containerPort: 8080

# EKS — AWS Load Balancer Controller 가 Ingress / Service 자동
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    alb.ingress.kubernetes.io/target-type: ip   # awsvpc
```

## 9. Multi-Target Group per Service

```yaml
# ECS — max 5 target group per service
loadBalancers:
  - targetGroupArn: tg-api-http
    containerName: api
    containerPort: 8080
  - targetGroupArn: tg-api-grpc
    containerName: api
    containerPort: 9090
  - targetGroupArn: tg-admin
    containerName: admin
    containerPort: 8081
```

## 10. Cross-zone load balancing

```bash
aws elbv2 modify-load-balancer-attributes --load-balancer-arn $ALB_ARN \
  --attributes Key=load_balancing.cross_zone.enabled,Value=true
```

| | ALB | NLB |
| --- | --- | --- |
| Cross-zone | **free + default** | $0.01/GB |

→ ALB 권장 (NLB 와 비용 큰 차이).

## 11. ALB-as-Target (Chain)

```bash
# Inner ALB 의 TG
aws elbv2 create-target-group --name tg-inner-alb \
  --target-type alb --port 80 --protocol HTTP \
  --vpc-id $VPC_ID

aws elbv2 register-targets --target-group-arn $TG_ALB \
  --targets Id=$INNER_ALB_ARN

# Outer NLB → Inner ALB (static IP + HTTP routing)
```

활용: NLB 의 static IP + ALB 의 L7 routing 결합.

## 12. CDK 예

```typescript
const alb = new elbv2.ApplicationLoadBalancer(this, 'Alb', {
  vpc,
  internetFacing: true,
  securityGroup: sgAlb,
});

const listener = alb.addListener('Https', {
  port: 443,
  protocol: elbv2.ApplicationProtocol.HTTPS,
  certificates: [cert],
  sslPolicy: elbv2.SslPolicy.TLS13_RES,
  defaultAction: elbv2.ListenerAction.fixedResponse(404),
});

listener.addAction('Api', {
  conditions: [elbv2.ListenerCondition.pathPatterns(['/api/*'])],
  priority: 10,
  action: elbv2.ListenerAction.forward([apiTg]),
});
```

## 13. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Target type instance + awsvpc | ip 사용 |
| Health Check grace period 짧음 | 120+ |
| Deregistration delay 길음 (default 300) | 30 권장 |
| Cross-zone OFF (NLB) | enable |
| Lambda Target permission 누락 | add-permission |
| Multi-TG 5 한도 | service 분리 |

## 14. 다음

- ▶ [[02-routing-rules-and-conditions]]
- ▶ [[03-pricing-and-limits]]
- ▶ [[../02-features/01-https-and-acm]]

## 15. 영상 / 자료

| 자료 |
| --- |
| [ALB Listener](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html) |
| [Target Type 비교](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html#target-type) |

## 16. 참고

- [[../alb]]
- [[../../../compute/ecs/03-service/05-load-balancer-integration]]
- [[../../../compute/eks/02-workload/02-service-and-ingress]]
