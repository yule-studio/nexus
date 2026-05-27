---
title: "NLB 01-1 — Listener + Target Group (TCP / UDP / TLS)"
kind: knowledge
status: current
created_at: 2026-05-23T10:30:00+09:00
tags: [aws, nlb, fundamentals, listener, target-group, tcp, udp, tls]
home_hub: nlb
---

# NLB Listener + Target Group

> TCP / UDP / TLS / TCP_UDP listener + Instance / IP / ALB target. Source IP 보존 + health check.

## 1. NLB 생성

```bash
NLB_ARN=$(aws elbv2 create-load-balancer --name bkurs-nlb \
  --type network \
  --scheme internet-facing \
  --subnet-mappings SubnetId=$SUBNET_A,AllocationId=$EIP_A \
                    SubnetId=$SUBNET_B,AllocationId=$EIP_B \
                    SubnetId=$SUBNET_C,AllocationId=$EIP_C \
  --ip-address-type ipv4 \
  --tags Key=Environment,Value=prod \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)
```

| 옵션 | 의미 |
| --- | --- |
| `--type network` | NLB |
| `--scheme` | internet-facing / internal |
| `--subnet-mappings` | EIP 명시 가능 (static IP per AZ) |
| `--ip-address-type` | ipv4 / dualstack |

## 2. Listener 4 종

### TCP
```bash
aws elbv2 create-listener --load-balancer-arn $NLB_ARN \
  --protocol TCP --port 5432 \
  --default-actions Type=forward,TargetGroupArn=$TG_POSTGRES
```

### UDP
```bash
aws elbv2 create-listener --load-balancer-arn $NLB_ARN \
  --protocol UDP --port 53 \
  --default-actions Type=forward,TargetGroupArn=$TG_DNS
```

### TLS (Termination)
```bash
aws elbv2 create-listener --load-balancer-arn $NLB_ARN \
  --protocol TLS --port 443 \
  --certificates CertificateArn=$ACM \
  --ssl-policy ELBSecurityPolicy-TLS13-1-2-2021-06 \
  --alpn-policy HTTP2Preferred \
  --default-actions Type=forward,TargetGroupArn=$TG_API
```

### TCP_UDP (DNS, RADIUS)
```bash
aws elbv2 create-listener --load-balancer-arn $NLB_ARN \
  --protocol TCP_UDP --port 53 \
  --default-actions Type=forward,TargetGroupArn=$TG_DNS
```

## 3. Target Group 3 종

| Type | Use |
| --- | --- |
| **instance** | EC2 — Source IP 보존 X (Disable PreserveClientIp 시) |
| **ip** | Fargate / EKS — Source IP 보존 가능 |
| **ALB** | NLB → ALB chain |

```bash
# TG TCP/IP type
TG_POSTGRES=$(aws elbv2 create-target-group --name bkurs-postgres \
  --protocol TCP --port 5432 \
  --vpc-id $VPC_ID --target-type ip \
  --health-check-protocol TCP --health-check-port 5432 \
  --healthy-threshold-count 3 --unhealthy-threshold-count 3 \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

# TG TLS
TG_API_TLS=$(aws elbv2 create-target-group --name bkurs-api-tls \
  --protocol TLS --port 8443 \
  --vpc-id $VPC_ID --target-type ip \
  --health-check-protocol HTTPS --health-check-path /health \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

# TG ALB (NLB → ALB chain)
TG_ALB=$(aws elbv2 create-target-group --name bkurs-tg-alb \
  --target-type alb --port 80 --protocol TCP \
  --vpc-id $VPC_ID \
  --health-check-protocol HTTP --health-check-port 80 --health-check-path /health \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

aws elbv2 register-targets --target-group-arn $TG_ALB \
  --targets Id=$INNER_ALB_ARN
```

## 4. TG attributes

```bash
aws elbv2 modify-target-group-attributes --target-group-arn $TG_ARN \
  --attributes \
    Key=deregistration_delay.timeout_seconds,Value=30 \
    Key=preserve_client_ip.enabled,Value=true \
    Key=proxy_protocol_v2.enabled,Value=false \
    Key=stickiness.enabled,Value=false
```

| Attribute | 의미 |
| --- | --- |
| `deregistration_delay.timeout_seconds` | drain (default 300) |
| `preserve_client_ip.enabled` | Source IP 보존 (IP target 만, instance target 은 default true) |
| `proxy_protocol_v2.enabled` | Proxy Protocol v2 (Source IP 전달 헤더) |
| `stickiness.enabled` | source IP sticky (TCP/TLS only) |
| `target_health_state.unhealthy.connection_termination.enabled` | unhealthy 시 connection 종료 |

## 5. Health Check

```bash
aws elbv2 modify-target-group --target-group-arn $TG_ARN \
  --health-check-protocol HTTP \
  --health-check-port 8080 \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --health-check-timeout-seconds 10 \
  --healthy-threshold-count 3 \
  --unhealthy-threshold-count 3
```

| Protocol | Use |
| --- | --- |
| TCP | basic port open check |
| HTTP / HTTPS | application level |
| HTTPS | with cert validation |

> NLB 의 health check 출발지 = NLB 의 ENI (각 AZ). Target SG inbound = NLB SG (또는 subnet CIDR).

## 6. Source IP 보존

```
NLB IP target + preserve_client_ip=true:
  Client (203.0.113.10) → NLB → Target 이 보는 source IP = 203.0.113.10

vs ALB / Proxy Protocol off NLB:
  Client (203.0.113.10) → NLB IP → Target 이 보는 source IP = NLB
  X-Forwarded-For header 로만
```

→ NLB 의 큰 장점 — application 이 진짜 client IP 직접.

## 7. Proxy Protocol v2

```bash
aws elbv2 modify-target-group-attributes --target-group-arn $TG_ARN \
  --attributes Key=proxy_protocol_v2.enabled,Value=true
```

Target 의 application 이 PP v2 header 처리:
```
PROXY TCP4 203.0.113.10 10.0.10.5 54321 8080\r\n
```

→ Layer 4 의 metadata 전달 (NLB IP target 외에도 활용).

## 8. ALPN Policy (TLS Listener)

```bash
--alpn-policy HTTP2Preferred
# 또는 HTTP2Only / HTTP2Optional / None
```

→ HTTP/2 의 NLB 통한 pass-through (gRPC over NLB).

## 9. Register Targets

```bash
# Instance
aws elbv2 register-targets --target-group-arn $TG \
  --targets Id=i-xxx,Port=8080 Id=i-yyy,Port=8080

# IP
aws elbv2 register-targets --target-group-arn $TG \
  --targets Id=10.0.10.5,Port=8080 Id=10.0.11.5,Port=8080

# ALB
aws elbv2 register-targets --target-group-arn $TG_ALB \
  --targets Id=arn:aws:elasticloadbalancing:...:loadbalancer/app/inner/abc

# Cross-AZ IP (다른 region's CIDR — Direct Connect)
aws elbv2 register-targets --target-group-arn $TG \
  --targets Id=172.16.0.10,Port=8080 \
  --availability-zone all
```

## 10. Cross-zone — NLB 특별

```bash
aws elbv2 modify-load-balancer-attributes --load-balancer-arn $NLB_ARN \
  --attributes Key=load_balancing.cross_zone.enabled,Value=true
```

| | Default | Cost |
| --- | --- | --- |
| Cross-zone OFF | per-AZ Stickiness | DT $0 |
| Cross-zone ON | 모든 AZ 균등 분산 | DT $0.01/GB |

> ALB 는 free + default ON. NLB 는 OFF + 비용. **Health check 의 false negative 시 cross-zone ON 권장**.

## 11. NLB Health Check from outside subnet

```
Health check 출발지:
- NLB IP (subnet CIDR) — 보통
- 또는 health check 의 source = 그 subnet 의 NLB ENI

SG / NACL allow rule 필요
```

## 12. CDK 예

```typescript
const nlb = new elbv2.NetworkLoadBalancer(this, 'Nlb', {
  vpc,
  internetFacing: true,
  crossZoneEnabled: true,
});

const listener = nlb.addListener('Tcp', { port: 5432 });
const tg = listener.addTargets('Postgres', {
  port: 5432,
  targets: [new elbv2_targets.IpTarget('10.0.10.5')],
  preserveClientIp: true,
  healthCheck: { protocol: elbv2.Protocol.TCP },
});
```

## 13. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Source IP 보존 가정 (Instance target) | IP target + preserve_client_ip |
| Cross-zone OFF + AZ 불균등 부하 | enable + 비용 인지 |
| Lambda target | ALB 만 |
| L7 routing | ALB |
| Health Check fail 시 service down | cross-zone + multi-AZ |
| TLS termination + backend HTTP | HTTPS 또는 PrivateLink |

## 14. 다음

- ▶ [[02-static-ip-and-cross-zone]]
- ▶ [[03-pricing-and-limits]]
- ▶ [[../02-features/01-tls-and-mtls]]

## 15. 영상 / 자료

| 자료 |
| --- |
| [NLB 공식](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/) |
| [NLB Target Type](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-target-groups.html#target-type) |

## 16. 참고

- [[../nlb]]
- [[../../alb/alb]]
