---
title: "AWS ALB / NLB — Load Balancer"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:05:00+09:00
tags:
  - aws
  - network
  - alb
  - nlb
  - load-balancer
---

# AWS ALB / NLB

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | ALB / NLB 비교 + 사용 |

**[[network|↑ Network]]** · **[[cloud-aws|↑↑ AWS]]**

---

## 1. 종류

| Type | OSI Layer | 용도 |
| --- | --- | --- |
| **ALB** (Application LB) | L7 (HTTP) | 일반 웹 / API |
| **NLB** (Network LB) | L4 (TCP/UDP) | 초저 latency / 정적 IP |
| **GWLB** (Gateway LB) | L3 | 보안 어플라이언스 chain |
| **CLB** (Classic LB) | L4 + L7 | 옛 — 사용 X |

→ 대부분 **ALB**. 게임 / VoIP / TCP 자체 = **NLB**.

---

## 2. ALB

### 2.1 핵심
- L7 routing (path / host / header)
- HTTP/2, gRPC, WebSocket
- TLS termination
- WAF 통합
- target = EC2 / IP / Lambda / ECS

### 2.2 생성

```hcl
resource "aws_lb" "alb" {
  name               = "myapp-alb"
  load_balancer_type = "application"
  subnets            = module.vpc.public_subnets
  security_groups    = [aws_security_group.alb.id]

  enable_deletion_protection = true
  enable_http2               = true
}

resource "aws_lb_target_group" "app" {
  name        = "myapp-tg"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = module.vpc.vpc_id
  target_type = "ip"        # 또는 instance, lambda

  health_check {
    path                = "/health"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 3
  }
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.alb.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.cert.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.alb.arn
  port              = 80
  protocol          = "HTTP"
  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}
```

### 2.3 Path-based routing

```hcl
resource "aws_lb_listener_rule" "api" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 100

  condition {
    path_pattern { values = ["/api/*"] }
  }
  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }
}
```

### 2.4 Sticky session

```hcl
resource "aws_lb_target_group" "app" {
  stickiness {
    type            = "lb_cookie"
    cookie_duration = 3600
    enabled         = true
  }
}
```

---

## 3. NLB

### 3.1 핵심
- L4 (TCP / UDP / TLS)
- 초저 latency
- **고정 IP** (각 AZ 마다 elastic IP)
- 매 초 수백만 연결
- WebSocket / 게임 / RTSP / VoIP

### 3.2 생성

```hcl
resource "aws_lb" "nlb" {
  name               = "myapp-nlb"
  load_balancer_type = "network"
  subnets            = module.vpc.public_subnets
}

resource "aws_lb_target_group" "tcp" {
  name        = "myapp-tcp-tg"
  port        = 8080
  protocol    = "TCP"
  vpc_id      = module.vpc.vpc_id
  target_type = "ip"
}

resource "aws_lb_listener" "tcp" {
  load_balancer_arn = aws_lb.nlb.arn
  port              = 443
  protocol          = "TLS"
  certificate_arn   = aws_acm_certificate.cert.arn
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tcp.arn
  }
}
```

---

## 4. ALB vs NLB

| 항목 | ALB | NLB |
| --- | --- | --- |
| Layer | L7 HTTP | L4 TCP/UDP |
| Latency | 수십 ms 추가 | μ초 |
| 처리량 | 매우 큼 | 더 큼 |
| 고정 IP | X (DNS) | ✅ |
| Path/host routing | ✅ | X |
| WebSocket | ✅ | ✅ |
| WAF / OIDC | ✅ | X |
| 가격 | LCU 기반 | NLB 기반 (Throughput) |

---

## 5. 헬스체크

ALB:
```
HTTP GET /health → 2xx/3xx OK
interval 30s, timeout 5s, threshold 2 healthy / 3 unhealthy
```

NLB:
```
TCP connect 또는 HTTP
```

→ 응용 안에 `/health` endpoint 작성. DB / cache 까지 확인 옵션.

---

## 6. ACM (TLS 인증서)

```hcl
resource "aws_acm_certificate" "cert" {
  domain_name       = "*.example.com"
  validation_method = "DNS"

  subject_alternative_names = ["example.com"]
}

resource "aws_route53_record" "cert_validation" {
  for_each = { for d in aws_acm_certificate.cert.domain_validation_options : d.domain_name => {...} }
  ...
}
```

→ DNS validation → 자동 renewal. 무료.
자세히 → [[security#3-추가-서비스-별도-노트-예정]]

---

## 7. WAF 통합

```hcl
resource "aws_wafv2_web_acl_association" "alb" {
  resource_arn = aws_lb.alb.arn
  web_acl_arn  = aws_wafv2_web_acl.main.arn
}
```

→ SQL injection / XSS / rate limit / IP block.

---

## 8. 비용 (Seoul, 대략)

ALB:
```
LCU = New connections + Active connections + Processed bytes + Rule evaluations
~ $0.0252 / 시간 + LCU 비용
대략 $22/월 + 트래픽
```

NLB:
```
NLCU = TCP/UDP connections + Bytes processed
~ $0.0252 / 시간 + NLCU
```

CloudFront 앞단 = ALB cost 감소 (cache hit).

---

## 9. 사용 시나리오

- 일반 웹 / API → **ALB**
- 마이크로서비스 (path routing) → ALB
- WebSocket → ALB / NLB 둘 다
- gRPC → ALB (gRPC support)
- TCP / UDP 게임 → NLB
- 고정 IP 필요 (legacy 통신) → NLB
- Lambda backend → ALB (target=lambda)

---

## 10. 함정

### 10.1 ALB 의 idle timeout
기본 60 초. 긴 응답 / long polling = 늘리기 (max 4000s).

### 10.2 health check failing
SG / NACL / path / port mismatch. ALB 의 SG 가 target SG 에 접근 가능해야.

### 10.3 cross-zone load balancing
ALB = 항상 ON. NLB = 옵션 (ON 권장 + cross-AZ DT 비용 주의).

### 10.4 SSL Policy
옛 (TLS 1.0) 사용 X. `ELBSecurityPolicy-TLS13-1-2-2021-06` 등.

### 10.5 deletion protection
운영 LB = enable.

### 10.6 path-based 우선순위
priority 작을수록 우선. default rule 은 마지막.

### 10.7 NLB 의 source IP preservation
target_type=instance → 클라이언트 IP 보존. ip 모드 = LB IP 보임 (X-Forwarded-For X).

---

## 11. 학습 자료

- AWS ELB docs
- **ALB Workshop**

---

## 12. 관련

- [[network]] — Network hub
- [[cloudfront]] — 앞단 CDN
- [[route53]] — DNS
- [[security]] — WAF / ACM
- [[vpc]] — subnet
