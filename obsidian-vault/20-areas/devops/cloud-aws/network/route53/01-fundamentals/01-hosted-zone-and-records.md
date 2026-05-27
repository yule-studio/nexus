---
title: "Route 53 01-1 — Hosted Zone (Public/Private) + Records + Alias"
kind: knowledge
status: current
created_at: 2026-05-23T11:22:00+09:00
tags: [aws, route53, fundamentals, hosted-zone, records, alias]
home_hub: route53
---

# Route 53 Hosted Zone + Records

> Public Hosted Zone (internet) vs Private Hosted Zone (VPC). Alias (AWS resource, free) vs CNAME. Record types.

## 1. Public Hosted Zone

```bash
aws route53 create-hosted-zone \
  --name bkurs.com \
  --caller-reference bkurs-prod-$(date +%s) \
  --hosted-zone-config "Comment=bkurs prod,PrivateZone=false"

HZ_ID=$(aws route53 list-hosted-zones-by-name --dns-name bkurs.com \
  --query 'HostedZones[0].Id' --output text | sed 's|/hostedzone/||')

# NS records (registrar 에 등록 필요)
aws route53 get-hosted-zone --id $HZ_ID \
  --query 'DelegationSet.NameServers'
```

→ Public DNS. Internet 의 모든 resolver 가 query.

## 2. Private Hosted Zone (VPC only)

```bash
aws route53 create-hosted-zone \
  --name bkurs.internal \
  --caller-reference bkurs-internal-$(date +%s) \
  --hosted-zone-config "PrivateZone=true" \
  --vpc "VPCRegion=ap-northeast-2,VPCId=$VPC_ID"

# 추가 VPC 연결
aws route53 associate-vpc-with-hosted-zone --hosted-zone-id $HZ_ID \
  --vpc "VPCRegion=ap-northeast-1,VPCId=$VPC_TOKYO"
```

→ VPC 안에서만 resolve. Internet 안 보임.

| 활용 |
| --- |
| DB endpoint 의 friendly name |
| Internal service discovery |
| Multi-environment (`dev.bkurs.internal`) |

## 3. Records 생성

```bash
aws route53 change-resource-record-sets --hosted-zone-id $HZ_ID \
  --change-batch '{
    "Changes": [
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "api.bkurs.com.",
          "Type": "A",
          "TTL": 60,
          "ResourceRecords": [{ "Value": "203.0.113.10" }]
        }
      },
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "cname.bkurs.com.",
          "Type": "CNAME",
          "TTL": 300,
          "ResourceRecords": [{ "Value": "api.bkurs.com." }]
        }
      },
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "bkurs.com.",
          "Type": "MX",
          "TTL": 3600,
          "ResourceRecords": [
            { "Value": "10 mx1.bkurs.com." },
            { "Value": "20 mx2.bkurs.com." }
          ]
        }
      }
    ]
  }'
```

## 4. Alias Record (AWS resource)

```bash
# CloudFront alias (root domain)
aws route53 change-resource-record-sets --hosted-zone-id $HZ_ID \
  --change-batch "{
    \"Changes\": [{
      \"Action\": \"UPSERT\",
      \"ResourceRecordSet\": {
        \"Name\": \"bkurs.com.\",
        \"Type\": \"A\",
        \"AliasTarget\": {
          \"HostedZoneId\": \"Z2FDTNDATAQYW2\",
          \"DNSName\": \"d123.cloudfront.net.\",
          \"EvaluateTargetHealth\": false
        }
      }
    }]
  }"

# ALB alias
aws route53 change-resource-record-sets --hosted-zone-id $HZ_ID \
  --change-batch "{
    \"Changes\": [{
      \"Action\": \"UPSERT\",
      \"ResourceRecordSet\": {
        \"Name\": \"api.bkurs.com.\",
        \"Type\": \"A\",
        \"AliasTarget\": {
          \"HostedZoneId\": \"$ALB_HOSTED_ZONE\",
          \"DNSName\": \"bkurs-alb-xxx.ap-northeast-2.elb.amazonaws.com.\",
          \"EvaluateTargetHealth\": true
        }
      }
    }]
  }"
```

## 5. AWS Resource 의 Hosted Zone ID (Alias 의 `HostedZoneId`)

| Service | Hosted Zone ID |
| --- | --- |
| **CloudFront** | `Z2FDTNDATAQYW2` (글로벌 고정) |
| **S3 Static Website (Seoul)** | `Z3W03O7B5YMIYP` (region 별) |
| **ALB / NLB (Seoul)** | LB 마다 다름 (`describe-load-balancers --query 'LoadBalancers[0].CanonicalHostedZoneId'`) |
| **API Gateway (Seoul)** | region 별 |
| **Global Accelerator** | `Z2BJ6XQ5FK7U4H` (글로벌) |
| **Elastic Beanstalk (Seoul)** | `Z3JE5OI70TYKAS` (region 별) |

```bash
# ALB 의 hosted zone id
aws elbv2 describe-load-balancers --names bkurs-alb \
  --query 'LoadBalancers[0].CanonicalHostedZoneId' --output text
# Z3HK69MEXAMPLE
```

## 6. Alias vs CNAME

| | CNAME | Alias |
| --- | --- | --- |
| Apex (`bkurs.com`) | ✗ (DNS spec) | ✓ |
| Subdomain | ✓ | ✓ |
| AWS service 자동 (DNS 변동) | ✗ (manual) | ✓ |
| Cost | $0.40/M query | free |
| TTL | manual | AWS managed |
| EvaluateTargetHealth | ✗ | ✓ |

→ **AWS resource = Alias**. 외부 service = CNAME.

## 7. Record Types

### A / AAAA
```
A:    IPv4 (e.g. 203.0.113.10)
AAAA: IPv6 (e.g. 2001:db8::1)
```

### CNAME
```
www.bkurs.com → bkurs.com.
api.bkurs.com → app.k8s.cluster.com.
```

### MX (Mail)
```
bkurs.com → 10 mx1.google.com.
bkurs.com → 20 mx2.google.com.
```

> Lower priority = higher preference.

### TXT (SPF / DKIM / verification)
```
bkurs.com         → "v=spf1 include:_spf.google.com ~all"
_dmarc.bkurs.com  → "v=DMARC1; p=quarantine; rua=mailto:dmarc@bkurs.com"
default._domainkey.bkurs.com → "v=DKIM1; k=rsa; p=MIGfMA0GC..."
```

### CAA (Certificate Authority Authorization)
```
bkurs.com → "0 issue amazon.com"      # ACM 만 발급
bkurs.com → "0 issuewild letsencrypt.org"
bkurs.com → "0 iodef mailto:security@bkurs.com"
```

→ DNS 의 정책: 어떤 CA 가 cert 발급 가능. 침해된 다른 CA 의 무단 발급 차단.

### SRV (Service Discovery)
```
_xmpp-server._tcp.bkurs.com → 10 60 5269 xmpp1.bkurs.com.
                                ↑ ↑   ↑    ↑
                          priority weight port target
```

### NS / SOA
```
NS:  Hosted Zone 의 nameserver 들 (자동 4개)
SOA: zone 의 metadata (serial / refresh / retry / expire / minimum)
```

## 8. TTL

```
A 60s:       빠른 변경 (failover)
A 300s:      일반
CNAME 3600s: 자주 안 바뀜
MX 86400s:   거의 영구
```

> TTL 짧음 → DNS query cost ↑ / 변경 빠름
> TTL 길음 → query cost ↓ / 변경 늦음

## 9. Apex Domain 의 Alias (CloudFront)

```bash
# bkurs.com (apex) → CloudFront (CNAME 안 됨, Alias 만 OK)
aws route53 change-resource-record-sets --hosted-zone-id $HZ_ID \
  --change-batch "{
    \"Changes\": [{
      \"Action\": \"UPSERT\",
      \"ResourceRecordSet\": {
        \"Name\": \"bkurs.com.\",
        \"Type\": \"A\",
        \"AliasTarget\": {
          \"HostedZoneId\": \"Z2FDTNDATAQYW2\",
          \"DNSName\": \"d123.cloudfront.net.\",
          \"EvaluateTargetHealth\": false
        }
      }
    }]
  }"
```

> Apex 의 CNAME 안 됨 = DNS RFC. ANAME / Alias 가 우회.

## 10. CLI 활용

```bash
# Hosted Zone list
aws route53 list-hosted-zones --query 'HostedZones[].{Name:Name,Id:Id,Records:ResourceRecordSetCount}'

# Records list
aws route53 list-resource-record-sets --hosted-zone-id $HZ_ID \
  --query 'ResourceRecordSets[].{Name:Name,Type:Type,TTL:TTL,Value:ResourceRecords[].Value|[0]}'

# Dig test
dig +short api.bkurs.com
dig +short api.bkurs.com @8.8.8.8
dig +short api.bkurs.com TXT
dig +noall +answer api.bkurs.com
```

## 11. Bulk Import (Zone file)

```bash
# Zone file (BIND format)
cat > zone.txt <<'EOF'
$ORIGIN bkurs.com.
$TTL 3600

@      IN  A       203.0.113.10
www    IN  CNAME   bkurs.com.
api    IN  A       203.0.113.20
@      IN  MX  10  mx1.google.com.
@      IN  MX  20  mx2.google.com.
@      IN  TXT     "v=spf1 include:_spf.google.com ~all"
EOF

# AWS 의 GUI 또는 Terraform / CDK 권장
# (cli 의 zone file import 직접 안 됨)
```

## 12. Terraform

```hcl
resource "aws_route53_zone" "main" {
  name = "bkurs.com"
}

resource "aws_route53_record" "api" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.bkurs.com"
  type    = "A"

  alias {
    name                   = aws_lb.bkurs.dns_name
    zone_id                = aws_lb.bkurs.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "root" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "bkurs.com"
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.cdn.domain_name
    zone_id                = "Z2FDTNDATAQYW2"  # CloudFront 고정
    evaluate_target_health = false
  }
}
```

## 13. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Apex CNAME | Alias 사용 |
| AWS resource 의 CNAME (cost 발생) | Alias |
| TTL 너무 김 → failover 늦음 | 60-300s |
| MX priority 가 낮을수록 우선 | 10 / 20 |
| Private Hosted Zone 의 enableDnsHostnames 안 함 | VPC modify |
| NS records 의 NS 의 변경 | TTL 변경 후 wait |

## 14. 다음

- ▶ [[02-routing-policies]]
- ▶ [[03-pricing-and-limits]]
- ▶ [[../02-features/01-health-checks-and-failover]]

## 15. 영상 / 자료

| 자료 |
| --- |
| [Hosted Zone](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/AboutHZWorkingWith.html) |
| [Records](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/rrsets-working-with.html) |
| [Alias](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html) |

## 16. 참고

- [[../route53]]
- [[../../cloudfront/cloudfront]]
- [[../../alb/alb]]
