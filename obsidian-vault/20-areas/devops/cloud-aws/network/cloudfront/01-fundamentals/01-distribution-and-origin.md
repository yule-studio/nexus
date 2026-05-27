---
title: "CloudFront 01-1 — Distribution + Origin + Origin Group"
kind: knowledge
status: current
created_at: 2026-05-23T10:52:00+09:00
tags: [aws, cloudfront, fundamentals, distribution, origin, origin-group]
home_hub: cloudfront
---

# CloudFront Distribution + Origin

> Distribution 생성 + Origin (S3/ALB/Custom HTTP) + Origin Group (failover) + Cache Behavior.

## 1. Distribution 생성

```bash
DIST_ID=$(aws cloudfront create-distribution --distribution-config "$(cat <<EOF
{
  "CallerReference": "bkurs-prod-$(date +%s)",
  "Comment": "bkurs prod",
  "DefaultRootObject": "index.html",
  "Origins": {
    "Quantity": 1,
    "Items": [{
      "Id": "S3-bkurs-static",
      "DomainName": "bkurs-static.s3.ap-northeast-2.amazonaws.com",
      "OriginAccessControlId": "$OAC_ID",
      "S3OriginConfig": { "OriginAccessIdentity": "" }
    }]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "S3-bkurs-static",
    "ViewerProtocolPolicy": "redirect-to-https",
    "AllowedMethods": { "Quantity": 2, "Items": ["GET","HEAD"] },
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "OriginRequestPolicyId": "88a5eaf4-2fd4-4709-b370-b4c650ea3fcf",
    "Compress": true
  },
  "PriceClass": "PriceClass_200",
  "Enabled": true,
  "ViewerCertificate": {
    "ACMCertificateArn": "$CERT_ARN_US_EAST_1",
    "SSLSupportMethod": "sni-only",
    "MinimumProtocolVersion": "TLSv1.2_2021"
  },
  "WebACLId": "$WAF_ARN",
  "HttpVersion": "http2and3",
  "IsIPV6Enabled": true
}
EOF
)" --query 'Distribution.Id' --output text)

aws cloudfront wait distribution-deployed --id $DIST_ID
```

| 옵션 | 의미 |
| --- | --- |
| `PriceClass` | PriceClass_100 (US/EU) / 200 (+Asia) / All (전체) |
| `HttpVersion` | http1.1 / http2 / http2and3 |
| `IsIPV6Enabled` | IPv6 |
| `MinimumProtocolVersion` | TLSv1.2_2021 ★ |

## 2. Origin 종류

| Origin Type | 특징 |
| --- | --- |
| **S3 Bucket** | OAC + Bucket Policy |
| **S3 Static Website** | website endpoint (custom 401/403 등) |
| **ALB / NLB** | Custom Origin (HTTP/HTTPS) |
| **API Gateway** | Custom Origin |
| **EC2** | Custom Origin (public IP) |
| **Custom HTTP** | 외부 server (on-prem) |
| **MediaPackage** | Live streaming |
| **MediaStore** | VOD |

## 3. S3 Origin (OAC)

```bash
# OAC 생성
OAC_ID=$(aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "bkurs-s3-oac",
    "Description": "S3 OAC",
    "OriginAccessControlOriginType": "s3",
    "SigningBehavior": "always",
    "SigningProtocol": "sigv4"
  }' --query 'OriginAccessControl.Id' --output text)

# Distribution 의 Origin
"Origins": {
  "Items": [{
    "Id": "S3-bkurs-static",
    "DomainName": "bkurs-static.s3.ap-northeast-2.amazonaws.com",
    "OriginAccessControlId": "{OAC_ID}",
    "S3OriginConfig": { "OriginAccessIdentity": "" }
  }]
}

# S3 Bucket Policy
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "cloudfront.amazonaws.com" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::bkurs-static/*",
    "Condition": {
      "StringEquals": { "AWS:SourceArn": "arn:aws:cloudfront::ACCT:distribution/{DIST_ID}" }
    }
  }]
}
```

> S3 BPA + OAC + Bucket Policy = S3 private, CloudFront 만 access.

## 4. ALB Custom Origin

```json
"Origins": {
  "Items": [{
    "Id": "ALB-bkurs-api",
    "DomainName": "bkurs-alb-12345.ap-northeast-2.elb.amazonaws.com",
    "CustomOriginConfig": {
      "HTTPPort": 80,
      "HTTPSPort": 443,
      "OriginProtocolPolicy": "https-only",
      "OriginSslProtocols": {
        "Quantity": 1,
        "Items": ["TLSv1.2"]
      },
      "OriginReadTimeout": 30,
      "OriginKeepaliveTimeout": 5
    },
    "CustomHeaders": {
      "Quantity": 1,
      "Items": [
        { "HeaderName": "x-cf-secret", "HeaderValue": "secret-token-xxx" }
      ]
    }
  }]
}
```

### Custom Header — Origin 인증
```
Client → CloudFront → x-cf-secret 추가 → ALB
ALB 의 WAF rule: x-cf-secret 가 없으면 block

→ Internet 의 ALB 직접 우회 차단 (CloudFront 만 access)
```

## 5. Origin Group (Failover)

```json
"OriginGroups": {
  "Quantity": 1,
  "Items": [{
    "Id": "alb-failover-group",
    "FailoverCriteria": {
      "StatusCodes": {
        "Quantity": 4,
        "Items": [500, 502, 503, 504]
      }
    },
    "Members": {
      "Quantity": 2,
      "Items": [
        { "OriginId": "ALB-primary" },
        { "OriginId": "ALB-dr-region" }
      ]
    }
  }]
}
```

→ Primary 500/502/503/504 시 자동 secondary 로.

활용:
- DR (multi-region)
- Active-passive failover
- Maintenance window 자동

## 6. Origin Shield (옵션)

```json
"OriginShield": {
  "Enabled": true,
  "OriginShieldRegion": "ap-northeast-2"
}
```

→ Origin 앞 mid-tier cache. Cache miss 감소. Cost: $0.0075/10K req + DT.

활용:
- Origin 의 부하 큰 경우
- Live event (대규모 동시 access)
- 비싼 origin (EC2 / on-prem)

## 7. CDN Edge Cache Hierarchy

```
Client → Edge Location (600+) → Regional Edge Cache (13) → Origin

1. Edge Location (CDN POP):
   - 600+ 전 세계
   - 사용자 가까운 곳
   - Cache hit → 즉시 응답
   
2. Regional Edge Cache (mid-tier, 2-level cache):
   - 13 region (Tokyo / Seoul / etc)
   - Edge 의 cache miss 시 → Regional 확인
   - Regional 의 cache miss → Origin

3. Origin Shield (옵션):
   - 1 region (사용자 선택)
   - Origin 의 cache layer

4. Origin:
   - S3 / ALB / Custom
   - 모든 cache miss 의 최종
```

## 8. Multiple Origins

```json
"Origins": {
  "Items": [
    { "Id": "S3-static", "DomainName": "bkurs-static.s3.amazonaws.com", ... },
    { "Id": "ALB-api",   "DomainName": "bkurs-alb...", ... },
    { "Id": "S3-uploads", "DomainName": "bkurs-uploads.s3.amazonaws.com", ... }
  ]
}
```

Cache Behaviors:
```json
"CacheBehaviors": {
  "Items": [
    { "PathPattern": "/api/*",      "TargetOriginId": "ALB-api" },
    { "PathPattern": "/uploads/*",  "TargetOriginId": "S3-uploads" },
    { "PathPattern": "/static/*",   "TargetOriginId": "S3-static" }
  ]
}
```

## 9. Origin 의 Path

```json
"OriginPath": "/v1"
```

→ Origin 의 path prefix 자동 추가. e.g. `/index.html` → `/v1/index.html` on origin.

활용:
- Version 별 directory
- 다른 environment (dev / staging / prod) 같은 origin 안에서

## 10. Aliases (Custom Domain)

```json
"Aliases": {
  "Quantity": 3,
  "Items": ["bkurs.com", "www.bkurs.com", "static.bkurs.com"]
}
```

| 한도 | 100 aliases / distribution |

+ ACM cert (us-east-1) 의 same domain.

## 11. Default Root Object

```json
"DefaultRootObject": "index.html"
```

→ `https://bkurs.com/` → `index.html` 자동.

> S3 static website endpoint 와 다름. CloudFront 의 distribution level 기본값.

## 12. Custom Error Pages

```json
"CustomErrorResponses": {
  "Quantity": 2,
  "Items": [
    { "ErrorCode": 404, "ResponsePagePath": "/404.html", "ResponseCode": "404", "ErrorCachingMinTTL": 60 },
    { "ErrorCode": 403, "ResponsePagePath": "/403.html", "ResponseCode": "403", "ErrorCachingMinTTL": 60 }
  ]
}
```

→ SPA (React / Vue) 의 경우:
```json
{ "ErrorCode": 403, "ResponsePagePath": "/index.html", "ResponseCode": "200" }
```

→ S3 의 403 (file 없음) → index.html 응답 (React Router 처리).

## 13. HTTP/2 + HTTP/3

```json
"HttpVersion": "http2and3"
```

- HTTP/2 + HTTP/3 (QUIC over UDP) 둘 다
- Multiplexing + header compression
- Latency 추가 절감

## 14. CDK 예

```typescript
const dist = new cloudfront.Distribution(this, 'Cdn', {
  defaultBehavior: {
    origin: new origins.S3Origin(bucket, {
      originAccessIdentity: oai,  // 또는 OAC
    }),
    viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
    cachePolicy: cloudfront.CachePolicy.CACHING_OPTIMIZED,
    compress: true,
  },
  additionalBehaviors: {
    '/api/*': {
      origin: new origins.LoadBalancerV2Origin(alb),
      cachePolicy: cloudfront.CachePolicy.CACHING_DISABLED,
      originRequestPolicy: cloudfront.OriginRequestPolicy.ALL_VIEWER,
    },
  },
  domainNames: ['bkurs.com', 'www.bkurs.com'],
  certificate: cert,
  priceClass: cloudfront.PriceClass.PRICE_CLASS_200,
  enableIpv6: true,
  webAclId: wafAcl.attrArn,
});
```

## 15. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| ACM cert 의 region (ap-northeast-2) | us-east-1 |
| S3 Bucket Policy + OAC SourceArn 누락 | 필수 |
| Alias + cert SAN 불일치 | 일치 |
| Origin Shield 비용 인지 못 함 | needed 만 |
| SPA 의 client routing 안 됨 | CustomErrorResponses 403 → index.html |
| Default Root Object 빠짐 | 명시 |

## 16. 다음

- ▶ [[02-cache-behavior-and-policies]]
- ▶ [[03-pricing-and-limits]]
- ▶ [[../02-features/01-https-and-oac]]

## 17. 영상 / 자료

| 자료 |
| --- |
| [CloudFront Distribution](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/distribution-overview.html) |
| [Origin Group](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/high_availability_origin_failover.html) |
| [Origin Shield](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/origin-shield.html) |

## 18. 참고

- [[../cloudfront]]
- [[../../alb/alb]]
- [[../../../storage/s3/practice/01-static-site-cloudfront]]
