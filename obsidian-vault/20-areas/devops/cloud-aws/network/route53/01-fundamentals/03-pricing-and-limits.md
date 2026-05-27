---
title: "Route 53 01-3 — Pricing + Limits"
kind: knowledge
status: current
created_at: 2026-05-23T11:26:00+09:00
tags: [aws, route53, fundamentals, pricing, limits]
home_hub: route53
---

# Route 53 Pricing + Limits

> Hosted Zone + Query + Health Check + Traffic Policy + Domain Registration.

## 1. 가격 구성

```
1. Hosted Zone (월)
2. Query (per million)
3. Health Check (per check / 월)
4. Traffic Policy (per policy / 월)
5. Domain Registration (per domain / 년)
6. Resolver (rule / endpoint / query)
7. DNSSEC (KMS key 무료)
```

## 2. Hosted Zone

```
First 25 zones / account: $0.50/zone/월
Additional:               $0.10/zone/월

→ Small business: $1-2/월
→ Enterprise (100+ zones): ~$60/월
```

## 3. Standard Query

```
First 1B / 월:  $0.40/M
Tier 2 (next):  $0.20/M (less common)
```

### 1000 RPS sustained:
```
1000 × 86400 × 30 = 2.6B queries/월
First 1B × $0.40 = $400
Next 1.6B × $0.20 = $320
= $720/월
```

> 큰 traffic 의 DNS = 비용 무시 못 함.

## 4. Routing Policy Query

| Policy | $/M |
| --- | --- |
| **Standard / Alias** | $0.40 |
| **Latency** | $0.60 |
| **Geolocation / Geoproximity** | $0.70 |
| **IP-based** | $0.70 |
| **DNSSEC validation** | $0.50 |

### Alias 의 cost
```
Alias to AWS resource: free
  (CloudFront, ALB, S3, etc 의 query 비용 없음)
  
External CNAME: $0.40/M
```

→ **AWS resource 는 항상 Alias** = cost 절감.

## 5. Health Check

| Type | $/check/月 |
| --- | --- |
| **Basic (AWS endpoint, HTTP)** | $0.50 |
| **HTTPS / String matching** | $1.00 |
| **Advanced (String + Latency)** | $2.00 |
| **Calculated** | $1.00 |
| **CloudWatch Alarm-based** | $1.00 |

### Non-AWS endpoint (premium)
| | $/check/월 |
| --- | --- |
| Basic | $0.75 |
| HTTPS / String | $1.50 |
| Advanced | $3.00 |

## 6. Traffic Policy

```
$50 / policy version / 월
+ DNS query (위 metering)
```

> 일반 routing policy 면 충분. Traffic Policy 는 매우 복잡한 visual builder 필요 시만.

## 7. Domain Registration

```
.com:           $13/년
.net:           $11/년
.org:           $12/년
.io:            $45/년
.dev:           $13/년
.kr:            $13/년 (가능)
.co.kr:         $25/년 (private registrar 권장)
.cloud:         $32/년
.app:           $14/년
.ai:            $99/년

Annual auto-renewal
WHOIS Privacy:  무료 (대부분 TLD)
```

## 8. Resolver Pricing

```
Inbound Endpoint:  $0.125/h/ENI
Outbound Endpoint: $0.125/h/ENI
Resolver Query:    $0.40/M

DNS Firewall:      $0.60/M query (additional)
Resolver Rules:    무료
```

## 9. 비용 시뮬 — bkurs-be

```
Hosted Zone:
  bkurs.com (public):           $0.50
  bkurs.internal (private):     $0.50
  Total: $1/월

Query:
  10M query/월 (small) × $0.40/M = $4
  (Alias 의 query 는 무료)

Health Check:
  2 endpoint × $0.50 = $1
  
Domain:
  bkurs.com × $13/년 / 12 = $1.1

Total: ~$7-10/월
```

## 10. 비용 절감

```
1. Alias (CloudFront / ALB / S3) → query 무료
2. TTL ↑ → query ↓ (caching 더 길게)
3. Health Check 의 최소화 (alarm-based, calculated)
4. Domain 의 multi-year 결제 (할인)
5. Hosted Zone 25개 안 (premium 가격 X)
```

## 11. Hard Limits

| | |
| --- | --- |
| Hosted Zones / account | 500 (soft) |
| Records / Hosted Zone | 10000 (soft) |
| Health Checks / account | 200 (soft) |
| Domains / account | 20 (soft) |
| Record set Identifier per Name+Type | 100 |
| Geolocation 의 country/subdivision | total 245 |
| CIDR collections / account | 5 (soft) |
| CIDR locations / collection | 100 |
| CIDR per location | 1000 |
| Resolver Inbound Endpoint / region | 4 (soft) |
| Resolver Outbound Endpoint / region | 4 (soft) |
| Resolver Rules / region | 1000 (soft) |
| Resolver DNS Firewall rule groups | 500 (soft) |

## 12. Service Quotas 증액

```bash
aws service-quotas list-service-quotas --service-code route53 \
  --query "Quotas[?Adjustable==\`true\`].[QuotaName,Value]" --output table

aws service-quotas request-service-quota-increase \
  --service-code route53 \
  --quota-code L-4EA4796A \
  --desired-value 1000
```

## 13. CloudWatch metric

| Metric | 의미 |
| --- | --- |
| `HealthCheckStatus` | 0/1 |
| `HealthCheckPercentageHealthy` | regions 의 % |
| `ConnectionTime` | health check latency |
| `TimeToFirstByte` | HTTP 응답 |
| `SSLHandshakeTime` | TLS |
| `DNSQueries` (Resolver) | per VPC |

### Alarm

```bash
aws cloudwatch put-metric-alarm --alarm-name r53-healthcheck-fail \
  --metric-name HealthCheckStatus --namespace AWS/Route53 \
  --statistic Minimum --period 60 --evaluation-periods 3 \
  --threshold 1 --comparison-operator LessThanThreshold \
  --dimensions Name=HealthCheckId,Value=$HC_ID \
  --region us-east-1
```

> Route 53 metric region = **us-east-1**.

## 14. Cost Allocation Tag

```bash
aws route53 change-tags-for-resource --resource-type hostedzone \
  --resource-id $HZ_ID \
  --add-tags "Key=Environment,Value=prod" "Key=Service,Value=bkurs"
```

## 15. Free Tier (12개월)

```
첫 12개월 — 50 health checks 무료 (basic)
```

## 16. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| AWS resource 의 CNAME → query 비용 | Alias |
| TTL 60s + 큰 query rate → cost 폭주 | TTL 300+ |
| Health Check 너무 많음 | calculated / alarm 활용 |
| Domain renewal 자동 안 함 → 만료 | auto-renewal enable |
| Resolver Endpoint 24x7 cost | 필요 시 만 |
| Traffic Policy $50/月 | 일반 routing 으로 |

## 17. 다음

- ▶ [[../02-features/01-health-checks-and-failover]]
- ▶ [[../02-features/02-dnssec-and-resolver]]
- ▶ [[../practice/01-domain-and-routing]]

## 18. 영상 / 자료

| 자료 |
| --- |
| [Route 53 Pricing](https://aws.amazon.com/route53/pricing/) |
| [Route 53 Quotas](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/DNSLimitations.html) |

## 19. 참고

- [[../route53]]
- [[01-hosted-zone-and-records]]
- [[02-routing-policies]]
