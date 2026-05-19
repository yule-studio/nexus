---
title: "NLB — Network Load Balancer (L4)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T13:15:00+09:00
tags: [devops, cloud-aws, nlb, network]
---

# NLB — Network Load Balancer (L4)

**[[cloud-aws|↑ cloud-aws]]**

---

## 1. 무엇

```
Layer 4 (TCP/UDP) load balancer:
  - ALB 는 L7 (HTTP)
  - NLB 는 L4 (TCP / UDP / TLS)
  
특징:
  - 초당 수백만 connection
  - 매우 낮은 latency (ms 미만)
  - 고정 IP (per AZ)
  - PrivateLink endpoint 가능
  - TLS termination + passthrough
  - cross-zone (선택)

비용:
  - $0.0225/h ($16/mo) per LB
  - NLCU 사용량 ($)
```

---

## 2. ALB vs NLB

| | ALB | NLB |
| --- | --- | --- |
| Layer | L7 (HTTP/HTTPS) | L4 (TCP/UDP/TLS) |
| protocol | HTTP / HTTPS / gRPC / WebSocket | TCP / UDP / TLS |
| routing | path / host / header | IP + port |
| WAF | ✓ | ✗ (TLS termination 시만) |
| TLS | termination 만 | termination + passthrough |
| static IP | ✗ (DNS only) | ✓ (per AZ) |
| PrivateLink | ✗ | ✓ |
| cross-zone | default on | default off (별도 cost) |
| sticky | cookie | source IP |
| client IP | X-Forwarded-For | preserved (★) |
| latency | 약간 | 매우 낮음 |
| 비용 | LCU | NLCU |

→ **HTTP = ALB**. **TCP/UDP / 고정 IP / 매우 빠름 = NLB**.

---

## 3. 사용 사례

```
✓ NLB:
  - PostgreSQL / MySQL proxy
  - Redis / ElastiCache proxy
  - Kafka / RabbitMQ broker access
  - gaming server (UDP)
  - VoIP (SIP / RTP)
  - IoT (MQTT)
  - 매우 큰 throughput (수십만 connection)
  - 고정 IP 가 필요 (whitelisting)
  - PrivateLink endpoint
  - TLS passthrough (mTLS application)

✗ ALB 가 더 좋은 경우:
  - HTTP path / host routing
  - WAF 통합
  - cognito 인증
  - server-sent events
```

---

## 4. 생성 (Terraform)

```hcl
resource "aws_lb" "nlb" {
  name               = "main-nlb"
  internal           = false       # public
  load_balancer_type = "network"
  
  subnet_mapping {
    subnet_id     = var.subnet_a
    allocation_id = aws_eip.nlb_a.id    # 고정 IP
  }
  subnet_mapping {
    subnet_id     = var.subnet_b
    allocation_id = aws_eip.nlb_b.id
  }
  subnet_mapping {
    subnet_id     = var.subnet_c
    allocation_id = aws_eip.nlb_c.id
  }
  
  enable_cross_zone_load_balancing = true        # ★ 비용 있음
  enable_deletion_protection       = true
  
  access_logs {
    bucket  = aws_s3_bucket.access_logs.bucket
    prefix  = "nlb"
    enabled = true
  }
}

resource "aws_eip" "nlb_a" {
  domain = "vpc"
  tags   = {Name = "nlb-eip-a"}
}
```

→ EIP 으로 고정 IP. EIP 가 사용자 / 외부 system 의 whitelist 에.

---

## 5. Target Group

```hcl
resource "aws_lb_target_group" "postgres" {
  name        = "postgres-tg"
  port        = 5432
  protocol    = "TCP"
  vpc_id      = var.vpc_id
  target_type = "ip"                  # 또는 instance / lambda
  
  preserve_client_ip = true            # ★ 사용자 IP 보존
  
  health_check {
    enabled             = true
    protocol            = "TCP"        # 또는 HTTP (검증 더)
    port                = 5432
    healthy_threshold   = 3
    unhealthy_threshold = 3
    interval            = 30
  }
  
  stickiness {
    enabled = true
    type    = "source_ip"
  }
}

resource "aws_lb_target_group_attachment" "postgres" {
  count            = length(var.postgres_ips)
  target_group_arn = aws_lb_target_group.postgres.arn
  target_id        = var.postgres_ips[count.index]
  port             = 5432
}
```

---

## 6. Listener

```hcl
# TCP listener
resource "aws_lb_listener" "postgres" {
  load_balancer_arn = aws_lb.nlb.arn
  port              = "5432"
  protocol          = "TCP"
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.postgres.arn
  }
}

# TLS listener (termination)
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.nlb.arn
  port              = "443"
  protocol          = "TLS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.main.arn
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web.arn
  }
}

# UDP listener (gaming / VoIP)
resource "aws_lb_listener" "udp" {
  load_balancer_arn = aws_lb.nlb.arn
  port              = "5060"
  protocol          = "UDP"
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.voip.arn
  }
}

# TCP_UDP (DNS 등 둘 다 필요)
resource "aws_lb_listener" "dns" {
  load_balancer_arn = aws_lb.nlb.arn
  port              = "53"
  protocol          = "TCP_UDP"
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.dns.arn
  }
}
```

---

## 7. client IP preservation (★)

```
NLB default 동작:
  - instance target: client IP 보존 (자동)
  - IP target: NAT 됨 (NLB IP 로)
  
→ application 이 client IP 알려면:
  preserve_client_ip = true (IP target)
  또는 PROXY protocol v2

PROXY protocol v2:
  TCP payload 의 시작에 client 정보 prepend
  → application 이 parse 해야
```

```hcl
preserve_client_ip = true              # IP target 일 때만

# 또는 PROXY protocol
proxy_protocol_v2 = true
```

```
# nginx 에 PROXY protocol
listen 443 ssl proxy_protocol;
set_real_ip_from <NLB CIDR>;
real_ip_header proxy_protocol;
```

→ ALB 의 X-Forwarded-For 와 다른 방식.

---

## 8. cross-zone load balancing

```
default:
  ALB: ON (free)
  NLB: OFF ($ 추가 cross-AZ traffic)

OFF 시:
  3 AZ × 2 server / AZ = 6 server
  AZ a 의 NLB 가 AZ a 의 2 server 에만 분배
  → AZ b 의 client 라도 → AZ b 의 NLB → AZ b 의 server
  → 같은 AZ 안 (cheap)
  
ON 시:
  AZ a 의 NLB 가 6 server 모두 분배
  → cross-AZ 비용 ($0.01/GB × 2)
```

→ cost vs 균등 분배 trade-off.

→ 작은 traffic / cost 민감 = OFF. 큰 / 균등 필요 = ON.

---

## 9. PrivateLink (★)

```
NLB 가 VPC 의 endpoint 노출:
  - 다른 VPC / account 에 service 제공
  - public internet 안 거침
  
구조:
  Provider VPC:
    NLB → Service (endpoint service)
  
  Consumer VPC:
    VPC Endpoint (Interface) → NLB
    같은 region 의 어느 VPC 든
```

```hcl
# Provider: endpoint service
resource "aws_vpc_endpoint_service" "main" {
  acceptance_required        = false
  network_load_balancer_arns = [aws_lb.nlb.arn]
  
  allowed_principals = [
    "arn:aws:iam::222222222222:root"     # consumer account
  ]
}

# Consumer: endpoint
resource "aws_vpc_endpoint" "to_provider" {
  vpc_id              = var.consumer_vpc_id
  service_name        = "com.amazonaws.vpce.ap-northeast-2.vpce-svc-0abc"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.consumer_subnet_ids
  security_group_ids  = [aws_security_group.endpoint.id]
  private_dns_enabled = true
}
```

→ Snowflake / Datadog / 회사 internal service 가 PrivateLink 으로 제공.

→ 보안 + 비용 (egress 없음).

---

## 10. TLS termination vs passthrough

### A. TLS termination (NLB 에서 복호화)

```hcl
protocol        = "TLS"
certificate_arn = aws_acm_certificate.main.arn
ssl_policy      = "ELBSecurityPolicy-TLS13-1-2-2021-06"
```

→ NLB 가 SSL handshake. backend 는 TCP / HTTP (plain).

→ ACM 통합. 단순.

### B. TLS passthrough (NLB 가 안 복호화)

```hcl
protocol = "TCP"        # TLS 아님
port     = 443
```

→ application 이 직접 TLS 처리 (예: mTLS).

→ NLB 는 byte stream 만 forward.

---

## 11. health check

```
TCP health check:
  - TCP handshake 성공 여부만
  - application 의 health 모름
  
HTTP / HTTPS:
  - 더 정확 (application 응답 검증)
  - latency 약간 ↑
  - path / matcher 설정
```

```hcl
health_check {
  protocol            = "HTTP"
  port                = "8080"        # 별도 health check port 가능
  path                = "/actuator/health"
  matcher             = "200"
  interval            = 30
  timeout             = 10
  healthy_threshold   = 3
  unhealthy_threshold = 3
}
```

---

## 12. NLB + EKS

```yaml
# k8s Service 의 LoadBalancer
apiVersion: v1
kind: Service
metadata:
  name: my-tcp-svc
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-attributes: load_balancing.cross_zone.enabled=true
    service.beta.kubernetes.io/aws-load-balancer-target-group-attributes: preserve_client_ip.enabled=true
spec:
  type: LoadBalancer
  loadBalancerClass: service.k8s.aws/nlb
  selector: {app: my-tcp-app}
  ports:
    - {port: 5432, targetPort: 5432, protocol: TCP}
```

→ AWS Load Balancer Controller 가 자동 NLB + Target Group 만듦.

---

## 13. monitoring

```
CloudWatch metric:
  - ActiveFlowCount
  - NewFlowCount (per sec)
  - ProcessedBytes
  - TCP_ELB_Reset_Count
  - TCP_Target_Reset_Count
  - HealthyHostCount / UnHealthyHostCount
  - NewFlowCount_TLS / ClientTLSNegotiationErrorCount

alert:
  - UnHealthyHostCount > 0
  - TLS error count spike
  - latency p99 spike
```

---

## 14. access log

```
NLB 는 default flow log 만 (no application-level)

S3 enable:
  endpoint 별 connection
  client IP
  bytes transferred
  TLS detail
```

→ ALB 만큼 자세한 access log 는 없음. application log 와 결합.

---

## 15. troubleshoot

```bash
# health check fail
aws elbv2 describe-target-health \
    --target-group-arn arn:aws:elasticloadbalancing:...

# 또는
aws elbv2 describe-target-health \
    --target-group-arn ... \
    --targets Id=10.0.0.5,Port=5432

# 흔한 원인:
# 1. SG: backend SG 가 NLB CIDR 안 열림
#    (NLB 는 SG 없음 — backend SG 가 NLB 의 ENI IP 받음)
# 2. application 의 health endpoint fail
# 3. target type instance + Bastion network ACL
# 4. preserve_client_ip + return path 의 SG

# DNS resolve
dig my-nlb-xxx.elb.ap-northeast-2.amazonaws.com

# connection test
nc -zv my-nlb-xxx.elb.ap-northeast-2.amazonaws.com 5432
```

→ **NLB 는 SG 없음**. backend SG 가 NLB 의 traffic 받아야.

---

## 16. ★ NLB 의 hidden gotcha

```
1. SG 없음
   → backend SG 의 ingress rule 에 client CIDR (전체) 또는 NLB IP

2. client IP preservation (IP target)
   → return packet 의 source IP / port 가 client 와 동일
   → backend SG 가 client IP 의 source 로 보임
   → 정확한 rule 필요

3. idle timeout 350s (★ ALB 60s 와 다름)
   → long-running connection (DB / Kafka) 적합
   → keepalive 설정 350s 이하

4. AZ 의 NLB ENI 잃음 → 그 AZ traffic 끊김
   → Route53 health check + multi-LB 검토
```

---

## 17. cost 분석

```
LB hourly:    $0.0225/h × 24 × 30 = $16/mo
NLCU:         사용량 기반

NLCU 의 dim:
  - new connection: per sec
  - active connection: per minute
  - processed bytes: per GB
  - rule eval: per sec

ALB 보다 일반적으로 cheaper (large flow 일 때).
cross-zone enabled: + cross-AZ data transfer $$.
TLS termination: 추가 NLCU.
```

---

## 18. 함정

1. **SG 추가 시도** — NLB 는 SG 없음. backend SG 에.
2. **client IP preservation 의 return path** — backend SG 잘못 시 응답 안 옴.
3. **PROXY protocol 의 application 비호환** — nginx / HAProxy 만 native.
4. **TLS passthrough 의 ACM 미사용** — application 이 cert 관리.
5. **cross-zone off + AZ unbalanced** — 한 AZ 만 hot.
6. **EIP 변경** — whitelist 깨짐. EIP 영구 보존.
7. **WAF 필요** — NLB X. ALB 또는 CloudFront 앞에.
8. **TCP health check + application fail** — 못 잡음. HTTP 사용.
9. **idle timeout 350s 의 keepalive** — application 적정 설정.

---

## 19. 관련

- [[cloud-aws|↑ cloud-aws]]
- [[alb]]
- [[load-balancer-types|↗ LB types]]
- [[eks|↗ EKS]]
