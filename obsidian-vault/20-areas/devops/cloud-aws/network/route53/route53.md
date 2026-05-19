---
title: "AWS Route 53 — DNS + Traffic Policy"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:15:00+09:00
tags:
  - aws
  - network
  - dns
  - route53
---

# AWS Route 53

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Route 53 개념 + 사용 |

**[[network|↑ Network]]** · **[[cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

AWS 의 **DNS 서비스** + **도메인 등록** + **health check** + **트래픽 정책** (latency / weighted / failover).

이름 의미 — port 53 (DNS).

---

## 2. 왜

- 빠른 DNS (글로벌 anycast)
- AWS 자원 자동 통합 (ALB / CloudFront / S3)
- health check + DNS failover
- 트래픽 라우팅 정책 (lat / weight / geo)
- DNSSEC 지원

대안:
- Cloudflare DNS / NS1 / dnsimple
- 도메인 등록만 = Gandi / Namecheap

---

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Hosted Zone** | 도메인 (`example.com`) 의 record 집합 |
| **Public / Private Hosted Zone** | 인터넷 / VPC 내부 |
| **Record (RR)** | A / AAAA / CNAME / MX / TXT / NS / Alias |
| **Alias** | AWS 자원으로의 internal record (CNAME 비슷, root 도 가능) |
| **Health Check** | 외부 endpoint 모니터링 |
| **Routing Policy** | Simple / Weighted / Latency / Failover / Geo / Multi-value |
| **Domain Registration** | 도메인 직접 등록 |

---

## 4. 첫 zone

### 4.1 도메인 등록 (선택)

Route 53 → Registered domains → Register → `example.com` 검색.

```bash
aws route53domains register-domain --domain-name example.com ...
```

이미 외부에서 산 도메인은 NS 만 Route 53 으로 변경.

### 4.2 Hosted Zone 생성

```bash
aws route53 create-hosted-zone --name example.com --caller-reference $(date +%s)
# → 4 NS record. 도메인 registrar 에 등록.
```

### 4.3 Record 생성 (CLI)

```bash
aws route53 change-resource-record-sets --hosted-zone-id Z123456 --change-batch '
{
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "api.example.com",
      "Type": "A",
      "AliasTarget": {
        "HostedZoneId": "Z14GRHDCWA56QT",
        "DNSName": "myapp-alb-123.ap-northeast-2.elb.amazonaws.com",
        "EvaluateTargetHealth": true
      }
    }
  }]
}'
```

### 4.4 Terraform

```hcl
resource "aws_route53_zone" "main" {
  name = "example.com"
}

resource "aws_route53_record" "api" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"

  alias {
    name                   = aws_lb.alb.dns_name
    zone_id                = aws_lb.alb.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "www" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "www.example.com"
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.site.domain_name
    zone_id                = "Z2FDTNDATAQYW2"      # CloudFront fixed
    evaluate_target_health = false
  }
}
```

---

## 5. Alias vs CNAME

| | Alias | CNAME |
| --- | --- | --- |
| AWS 자원 | ✅ | ✅ (가능) |
| Root domain (`example.com`) | ✅ | ❌ |
| 비용 | 무료 (resolution) | 과금 |
| 응답 속도 | 빠름 | 추가 lookup |

→ AWS 자원 가리킬 땐 항상 **Alias**.

---

## 6. Routing Policy

### 6.1 Simple
기본 — 1 record, 1 또는 multi IP.

### 6.2 Weighted
```
api.example.com  →  10% → ALB v1
                 →  90% → ALB v2
```
Canary / blue-green deployment.

### 6.3 Latency
사용자의 latency 최소 region 의 endpoint.
```
api.example.com  →  Seoul ALB (한국 사용자)
                 →  Virginia ALB (미국 사용자)
```

### 6.4 Failover
```
Primary:   ALB (health check OK)
Secondary: S3 maintenance page (primary 실패 시)
```

### 6.5 Geolocation
국가 / 대륙 별 다른 endpoint.
```
KR  → Seoul ALB
JP  → Tokyo ALB
default → Virginia ALB
```

### 6.6 Multi-value
multiple healthy endpoint random return (DNS round-robin).

---

## 7. Health Check

```hcl
resource "aws_route53_health_check" "api" {
  fqdn              = "api.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30
}
```

→ failover routing 의 입력. CloudWatch 알람 통합.

---

## 8. Private Hosted Zone

```hcl
resource "aws_route53_zone" "internal" {
  name = "internal.example.com"
  vpc {
    vpc_id = module.vpc.vpc_id
  }
}
```

→ VPC 안에서만 resolve. service discovery / 내부 endpoint.

ECS Service Connect / Cloud Map 와 자주 함께.

---

## 9. DNSSEC

```bash
aws route53 enable-hosted-zone-dnssec --hosted-zone-id Z123
```

→ DNS spoofing 방어. KMS key 필요.

---

## 10. 가격 (Seoul)

```
Hosted Zone:     $0.50 / month (첫 25)
Queries:         $0.40 / 1M (첫 1B/월)
Alias:           무료 (resolution)
Health Check:    $0.50 / month
Domain register: TLD 별 ($12-$30/년)
DNSSEC:          + KMS key 비용
```

→ 일반적으로 매월 1-5 달러.

---

## 11. 사용 시나리오

- 도메인 + AWS 자원 매핑 (필수)
- multi-region failover
- canary deployment (weighted)
- service discovery (private zone)
- email (MX + SPF/DKIM TXT)
- DDoS / 글로벌 응답 — anycast 자체

---

## 12. 함정

### 12.1 NS 전파
도메인 registrar 에 새 NS 등록 → 24-48 시간 전파.

### 12.2 TTL 짧으면 비용 ↑
1 분 TTL = 매 분 query. 보통 300 초+.

### 12.3 CNAME root domain 불가
`example.com` 에 CNAME = RFC 위반. Alias 사용.

### 12.4 health check 만 의존
DNS TTL 동안 stale. 짧은 TTL + failover.

### 12.5 deletion
zone 삭제 전에 record 모두 제거.

### 12.6 latency routing 부정확
지구 위치 추정 ≠ 네트워크 latency. 측정해서 검증.

### 12.7 DNSSEC 의 rollback
잘못된 record / KMS 키 변경 = 사이트 다운. 신중.

---

## 13. 학습 자료

- AWS Route 53 docs
- **DNS 자체** — [[dns|↗ DNS hub]]

---

## 14. 관련

- [[network]] — Network hub
- [[alb]] / [[cloudfront]] — Alias target
- [[dns|↗ DNS 이론]]
