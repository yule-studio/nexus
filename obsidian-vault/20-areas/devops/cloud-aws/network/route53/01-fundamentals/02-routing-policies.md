---
title: "Route 53 01-2 — 8 Routing Policies"
kind: knowledge
status: current
created_at: 2026-05-23T11:24:00+09:00
tags: [aws, route53, fundamentals, routing-policy, weighted, latency, failover, geo]
home_hub: route53
---

# Route 53 8 Routing Policies

> Simple / Weighted / Latency / Failover / Geolocation / Geoproximity / Multi-value / IP-based.

## 1. 비교

| Policy | 의미 | Use Case |
| --- | --- | --- |
| **Simple** | 1 record (또는 multi-value) | 단일 endpoint |
| **Weighted** | weight 별 분산 (e.g. 90/10) | Blue-Green / Canary / A/B test |
| **Latency** | client 의 가까운 region | Multi-region performance |
| **Failover** | Active + Passive | DR |
| **Geolocation** | client 의 country/region | Region-specific content |
| **Geoproximity** | client geographic distance + bias | Geographic load balance (Traffic Policy 만) |
| **Multi-value Answer** | 여러 IP 응답 (with health check) | Simple 의 client-side LB |
| **IP-based** | client IP CIDR 별 | ISP / CDN specific |

## 2. Simple Routing

```bash
aws route53 change-resource-record-sets --hosted-zone-id $HZ_ID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.bkurs.com.",
        "Type": "A",
        "TTL": 60,
        "ResourceRecords": [{ "Value": "203.0.113.10" }]
      }
    }]
  }'
```

→ 1 record. Health check 없음.

## 3. Weighted Routing

```bash
# Blue (90%)
aws route53 change-resource-record-sets --hosted-zone-id $HZ_ID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.bkurs.com.",
        "Type": "A",
        "SetIdentifier": "blue",
        "Weight": 90,
        "TTL": 60,
        "ResourceRecords": [{ "Value": "203.0.113.10" }]
      }
    }]
  }'

# Green (10%)
aws route53 change-resource-record-sets --hosted-zone-id $HZ_ID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.bkurs.com.",
        "Type": "A",
        "SetIdentifier": "green",
        "Weight": 10,
        "TTL": 60,
        "ResourceRecords": [{ "Value": "203.0.113.20" }]
      }
    }]
  }'
```

```
Weight 0:    routing 안 함 (drain pattern)
Weight 1-255: 비율 분산
```

활용: Blue-Green / Canary / A/B test.

## 4. Latency-based Routing

```bash
# Seoul ALB
aws route53 change-resource-record-sets --hosted-zone-id $HZ_ID \
  --change-batch "{
    \"Changes\": [{
      \"Action\": \"UPSERT\",
      \"ResourceRecordSet\": {
        \"Name\": \"api.bkurs.com.\",
        \"Type\": \"A\",
        \"SetIdentifier\": \"seoul\",
        \"Region\": \"ap-northeast-2\",
        \"AliasTarget\": {
          \"HostedZoneId\": \"$ALB_SEOUL_HZ\",
          \"DNSName\": \"bkurs-seoul.ap-northeast-2.elb.amazonaws.com.\",
          \"EvaluateTargetHealth\": true
        }
      }
    }]
  }"

# Tokyo ALB
aws route53 change-resource-record-sets --hosted-zone-id $HZ_ID \
  --change-batch "{
    \"Changes\": [{
      \"Action\": \"UPSERT\",
      \"ResourceRecordSet\": {
        \"Name\": \"api.bkurs.com.\",
        \"Type\": \"A\",
        \"SetIdentifier\": \"tokyo\",
        \"Region\": \"ap-northeast-1\",
        \"AliasTarget\": {
          \"HostedZoneId\": \"$ALB_TOKYO_HZ\",
          \"DNSName\": \"bkurs-tokyo.ap-northeast-1.elb.amazonaws.com.\",
          \"EvaluateTargetHealth\": true
        }
      }
    }]
  }"
```

→ Client 의 latency 가장 가까운 region 선택.

활용:
- Multi-region API
- Global user base
- Performance critical

## 5. Failover Routing

```bash
# Primary (Active)
aws route53 change-resource-record-sets --hosted-zone-id $HZ_ID \
  --change-batch "{
    \"Changes\": [{
      \"Action\": \"UPSERT\",
      \"ResourceRecordSet\": {
        \"Name\": \"api.bkurs.com.\",
        \"Type\": \"A\",
        \"SetIdentifier\": \"primary\",
        \"Failover\": \"PRIMARY\",
        \"HealthCheckId\": \"$HC_PRIMARY\",
        \"AliasTarget\": {
          \"HostedZoneId\": \"$ALB_SEOUL_HZ\",
          \"DNSName\": \"bkurs-primary.ap-northeast-2.elb.amazonaws.com.\",
          \"EvaluateTargetHealth\": true
        }
      }
    }]
  }"

# Secondary (Passive)
aws route53 change-resource-record-sets --hosted-zone-id $HZ_ID \
  --change-batch "{
    \"Changes\": [{
      \"Action\": \"UPSERT\",
      \"ResourceRecordSet\": {
        \"Name\": \"api.bkurs.com.\",
        \"Type\": \"A\",
        \"SetIdentifier\": \"secondary\",
        \"Failover\": \"SECONDARY\",
        \"AliasTarget\": {
          \"HostedZoneId\": \"$ALB_TOKYO_HZ\",
          \"DNSName\": \"bkurs-dr.ap-northeast-1.elb.amazonaws.com.\",
          \"EvaluateTargetHealth\": true
        }
      }
    }]
  }"
```

→ Primary 의 health check fail = Secondary 로 자동.

| Type | 의미 |
| --- | --- |
| `PRIMARY` | 정상 시 라우팅 |
| `SECONDARY` | Primary unhealthy 시만 |

## 6. Geolocation Routing

```bash
# Asia (KR / JP / etc) → Seoul
aws route53 change-resource-record-sets --hosted-zone-id $HZ_ID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.bkurs.com.",
        "Type": "A",
        "SetIdentifier": "asia",
        "GeoLocation": { "ContinentCode": "AS" },
        "TTL": 60,
        "ResourceRecords": [{ "Value": "203.0.113.10" }]
      }
    }]
  }'

# Korea 만 → 한국 specific endpoint
"GeoLocation": { "CountryCode": "KR" }

# Specific subdivision (US-CA)
"GeoLocation": { "CountryCode": "US", "SubdivisionCode": "CA" }

# Default (다른 어느 곳에도 안 맞으면)
"GeoLocation": { "CountryCode": "*" }
```

| 활용 | |
| --- | --- |
| 한국 사용자 → 한국어 site |
| EU 사용자 → GDPR-compliant |
| US 사용자 → CDN edge |
| 차단된 국가 → block (NXDOMAIN) |

## 7. Geoproximity (Traffic Policy 만)

```
Geographic distance based:
- Resource 의 위치 (lat/long 또는 AWS region)
- Client 와의 거리 계산
- Bias (-99 to +99) 로 영향
```

→ Console 의 Traffic Policy Builder 만. 일반 record set 안 됨.

## 8. Multi-value Answer

```bash
# 여러 IP, 각각 health check
for IP in 203.0.113.10 203.0.113.11 203.0.113.12; do
  aws route53 change-resource-record-sets --hosted-zone-id $HZ_ID \
    --change-batch "{
      \"Changes\": [{
        \"Action\": \"UPSERT\",
        \"ResourceRecordSet\": {
          \"Name\": \"api.bkurs.com.\",
          \"Type\": \"A\",
          \"SetIdentifier\": \"server-$IP\",
          \"MultiValueAnswer\": true,
          \"HealthCheckId\": \"$HC_$IP\",
          \"TTL\": 60,
          \"ResourceRecords\": [{ \"Value\": \"$IP\" }]
        }
      }]
    }"
done
```

→ DNS query = healthy IP 중 최대 8개 반환. Client 가 random 선택.

활용:
- ALB / NLB 없이 simple LB
- Health check 와 결합 (unhealthy IP 자동 제외)

## 9. IP-based Routing (CIDR collection)

```bash
# CIDR collection 생성
COLL=$(aws route53 create-cidr-collection --name corp-isps \
  --query 'Collection.Id' --output text)

aws route53 change-cidr-collection --id $COLL \
  --changes '[
    {
      "Action": "PUT",
      "LocationName": "kt-isp",
      "CidrList": ["218.236.0.0/16","119.193.0.0/16"]
    },
    {
      "Action": "PUT",
      "LocationName": "sk-isp",
      "CidrList": ["1.224.0.0/13","27.0.0.0/13"]
    }
  ]'

# Record 의 IP-based
aws route53 change-resource-record-sets --hosted-zone-id $HZ_ID \
  --change-batch "{
    \"Changes\": [{
      \"Action\": \"UPSERT\",
      \"ResourceRecordSet\": {
        \"Name\": \"api.bkurs.com.\",
        \"Type\": \"A\",
        \"SetIdentifier\": \"kt-users\",
        \"CidrRoutingConfig\": {
          \"CollectionId\": \"$COLL\",
          \"LocationName\": \"kt-isp\"
        },
        \"TTL\": 60,
        \"ResourceRecords\": [{ \"Value\": \"203.0.113.10\" }]
      }
    }]
  }"
```

→ KT 사용자 → 특정 endpoint. SK / LGU+ 사용자 → 다른 endpoint. ISP 별 optimization.

## 10. 정책 결합

```
Latency + Failover:
  - Primary: Seoul (latency) → unhealthy 시 Tokyo
  - Tokyo region 의 latency policy → unhealthy 시 다른 region

Weighted + Latency:
  - Seoul region: Blue 90% / Green 10% (Weighted)
  - Latency 가 Seoul / Tokyo 선택
```

## 11. EvaluateTargetHealth (Alias 의)

```
EvaluateTargetHealth: true:
  Alias target 의 health 가 unhealthy → Route 53 의 routing 자동 제외
  - ALB: target group 의 모든 healthy 없음
  - CloudFront: 별도 health check
  - S3: bucket 존재 여부
```

→ Failover policy 와 결합 시 핵심.

## 12. Traffic Policy (visual builder)

Console 만의 GUI:
```
Traffic Policy Version:
1. Select Failover record type
2. Add health check
3. Branch to: Weighted / Latency / Geo policy
4. Final endpoints
```

→ 복잡 routing 의 visual design. CLI 안 됨 (JSON document).

비용:
- $50/policy/月

## 13. CDN / API Gateway 와 결합 패턴

```
api.bkurs.com:
- Latency-based to nearest region:
  - Seoul: API GW + CloudFront in ap-northeast-2
  - Tokyo: API GW + CloudFront in ap-northeast-1
- Failover: 각 region 의 health check
- Weighted: Blue/Green per region

static.bkurs.com:
- Simple A record → CloudFront (single distribution, global)
```

## 14. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Apex CNAME (안 됨) | Alias |
| TTL 너무 김 → failover 늦음 | 60s |
| Weighted SetIdentifier 같음 | unique |
| Failover health check 없음 | HealthCheckId 명시 |
| Geo의 default 없음 → "no answer" 반환 | `*` default |
| Multi-value 의 8개 한도 | 더 필요 시 LB |
| CIDR 가 겹침 | first match |

## 15. 다음

- ▶ [[03-pricing-and-limits]]
- ▶ [[../02-features/01-health-checks-and-failover]]
- ▶ [[../02-features/02-dnssec-and-resolver]]

## 16. 영상 / 자료

| 자료 |
| --- |
| [Routing Policy](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html) |
| [Traffic Policy](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/traffic-policies.html) |

## 17. 참고

- [[../route53]]
- [[01-hosted-zone-and-records]]
