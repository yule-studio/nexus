---
title: "CloudFront 01-2 — Cache Behavior + Cache Policy + Origin Request Policy + Response Headers"
kind: knowledge
status: current
created_at: 2026-05-23T10:54:00+09:00
tags: [aws, cloudfront, fundamentals, cache-behavior, cache-policy, origin-request-policy]
home_hub: cloudfront
---

# CloudFront Cache Behavior + Policies

> Behavior (path-based routing) + 3 Policy (Cache / Origin Request / Response Headers) — 2020+ 표준.

## 1. 3 Policy 비교

| Policy | 의미 |
| --- | --- |
| **Cache Policy** | TTL + Cache Key (어떤 header/cookie/query 가 cache key 의 일부) |
| **Origin Request Policy** | Origin 으로 forward 할 header / cookie / query |
| **Response Headers Policy** | Client 응답 에 추가할 header (CORS / Security) |

## 2. Cache Behavior

```json
"DefaultCacheBehavior": {
  "TargetOriginId": "S3-bkurs-static",
  "ViewerProtocolPolicy": "redirect-to-https",
  "AllowedMethods": { "Quantity": 2, "Items": ["GET","HEAD"] },
  "CachedMethods":  { "Quantity": 2, "Items": ["GET","HEAD"] },
  "CachePolicyId":          "658327ea-f89d-4fab-a63d-7e88639e58f6",
  "OriginRequestPolicyId":  "88a5eaf4-2fd4-4709-b370-b4c650ea3fcf",
  "ResponseHeadersPolicyId":"60669652-455b-4ae9-85a4-c4c02393f86c",
  "Compress": true,
  "SmoothStreaming": false,
  "FieldLevelEncryptionId": "",
  "FunctionAssociations": {...},
  "LambdaFunctionAssociations": {...},
  "TrustedKeyGroups": {...}
}
```

| 옵션 | 의미 |
| --- | --- |
| `ViewerProtocolPolicy` | allow-all / redirect-to-https / https-only |
| `AllowedMethods` | GET+HEAD / +OPTIONS / +DELETE+PATCH+POST+PUT |
| `CachedMethods` | GET+HEAD only (cache target) |
| `Compress` | gzip/br compression |
| `FunctionAssociations` | CloudFront Functions |
| `LambdaFunctionAssociations` | Lambda@Edge |

## 3. Cache Policies — AWS Managed

| ID | Name | TTL | Cache Key |
| --- | --- | --- | --- |
| `658327ea-f89d-4fab-a63d-7e88639e58f6` | CachingOptimized | 1d (min 1s, max 1y) | URL only |
| `b2884449-e4de-46a7-ac36-70bc7f1ddd6d` | CachingOptimizedForUncompressedObjects | 1d | URL + Accept-Encoding |
| `4135ea2d-6df8-44a3-9df3-4b5a84be39ad` | CachingDisabled | 0 | n/a |
| `08627262-05a9-4f76-9ded-b50ca2e3a84f` | UseOriginCacheControlHeaders | Origin 의 Cache-Control 따라 | URL + Cache-Control header |

## 4. Custom Cache Policy

```bash
aws cloudfront create-cache-policy --cache-policy-config '{
  "Name": "bkurs-api-cache",
  "Comment": "API cache with auth",
  "DefaultTTL": 60,
  "MinTTL": 0,
  "MaxTTL": 3600,
  "ParametersInCacheKeyAndForwardedToOrigin": {
    "EnableAcceptEncodingGzip": true,
    "EnableAcceptEncodingBrotli": true,
    "HeadersConfig": {
      "HeaderBehavior": "whitelist",
      "Headers": {"Quantity": 2, "Items": ["Authorization","X-Api-Version"]}
    },
    "CookiesConfig": {
      "CookieBehavior": "whitelist",
      "Cookies": {"Quantity": 1, "Items": ["session-id"]}
    },
    "QueryStringsConfig": {
      "QueryStringBehavior": "whitelist",
      "QueryStrings": {"Quantity": 2, "Items": ["lang","theme"]}
    }
  }
}'
```

### Cache Key 의 영향
```
Cache Key = URL + (whitelist 된 header/cookie/query)

같은 URL 이라도 Authorization header 가 다르면 다른 cache entry
→ user 별 different cache (storage 큼)
→ user 무관 = no header 포함 (storage 작음, hit ↑)
```

## 5. Origin Request Policies — AWS Managed

| Name | 의미 |
| --- | --- |
| `AllViewer` | 모든 header/cookie/query origin 전달 |
| `AllViewerExceptHostHeader` | Host 제외 |
| `AllViewerAndCloudFrontHeaders-2022-06` | + CF-* header |
| `CORS-CustomOrigin` | CORS 의 OPTIONS 처리 |
| `CORS-S3Origin` | CORS S3 |
| `UserAgentRefererHeaders` | UA + Referer |
| `Elemental-MediaTailor-PersonalizedManifests` | MediaTailor |

## 6. Custom Origin Request Policy

```bash
aws cloudfront create-origin-request-policy --origin-request-policy-config '{
  "Name": "bkurs-api-origin",
  "Comment": "Forward auth + custom headers",
  "HeadersConfig": {
    "HeaderBehavior": "whitelist",
    "Headers": {"Quantity": 3, "Items": ["Authorization","User-Agent","CloudFront-Viewer-Country"]}
  },
  "CookiesConfig": {
    "CookieBehavior": "all"
  },
  "QueryStringsConfig": {
    "QueryStringBehavior": "all"
  }
}'
```

→ Cache Policy 보다 더 넓은 set 가능 (origin 에 전달은 하되, cache key 는 아님).

## 7. Response Headers Policies

```bash
aws cloudfront create-response-headers-policy --response-headers-policy-config '{
  "Name": "bkurs-security-headers",
  "Comment": "Security + CORS",
  "SecurityHeadersConfig": {
    "StrictTransportSecurity": {
      "Override": true,
      "AccessControlMaxAgeSec": 63072000,
      "IncludeSubdomains": true,
      "Preload": true
    },
    "ContentTypeOptions": { "Override": true },
    "FrameOptions": { "Override": true, "FrameOption": "DENY" },
    "ReferrerPolicy": { "Override": true, "ReferrerPolicy": "strict-origin-when-cross-origin" },
    "ContentSecurityPolicy": {
      "Override": true,
      "ContentSecurityPolicy": "default-src '\''self'\''; img-src '\''self'\'' data:"
    },
    "XSSProtection": { "Override": true, "Protection": true, "ModeBlock": true }
  },
  "CorsConfig": {
    "AccessControlAllowCredentials": false,
    "AccessControlAllowHeaders": {"Quantity": 1, "Items": ["*"]},
    "AccessControlAllowMethods":  {"Quantity": 4, "Items": ["GET","HEAD","OPTIONS","POST"]},
    "AccessControlAllowOrigins":  {"Quantity": 1, "Items": ["https://bkurs.com"]},
    "AccessControlExposeHeaders": {"Quantity": 1, "Items": ["x-request-id"]},
    "AccessControlMaxAgeSec": 600,
    "OriginOverride": true
  },
  "CustomHeadersConfig": {
    "Quantity": 1,
    "Items": [
      { "Header": "X-Bkurs-Region", "Value": "ap-northeast-2", "Override": true }
    ]
  },
  "RemoveHeadersConfig": {
    "Quantity": 2,
    "Items": [
      { "Header": "x-amzn-requestid" },
      { "Header": "x-amzn-trace-id" }
    ]
  }
}'
```

## 8. TTL 의 동작

```
Min TTL  ≤ TTL ≤ Max TTL
- Default TTL: Origin 의 Cache-Control 없으면 사용
- Origin 의 Cache-Control: 우선 (Min/Max 안)
```

### Origin Cache-Control 예
```
Cache-Control: max-age=3600    → CDN cache 1h
Cache-Control: no-cache        → 항상 revalidate
Cache-Control: no-store        → cache 안 함
Cache-Control: s-maxage=86400  → CDN 만 1d (browser 와 별도)
Cache-Control: private         → browser 만, CDN ✗
Cache-Control: public          → CDN + browser
Cache-Control: stale-while-revalidate=86400  → stale 응답 + background refresh
```

> CloudFront 가 `s-maxage` 우선 (CDN 전용). `max-age` = browser cache.

## 9. Behavior 의 Path Pattern

```json
"CacheBehaviors": {
  "Items": [
    {
      "PathPattern": "/api/*",
      "TargetOriginId": "ALB-api",
      "ViewerProtocolPolicy": "https-only",
      "AllowedMethods": { "Quantity": 7, "Items": ["GET","HEAD","OPTIONS","PUT","POST","PATCH","DELETE"] },
      "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad",  // CachingDisabled
      "OriginRequestPolicyId": "216adef6-5c7f-47e4-b989-5492eafa07d3"  // AllViewer
    },
    {
      "PathPattern": "/images/*",
      "TargetOriginId": "S3-bkurs-static",
      "CachePolicyId": "658327ea-..."  // CachingOptimized 1d
    },
    {
      "PathPattern": "/static/*",
      "TargetOriginId": "S3-bkurs-static",
      "CachePolicyId": "658327ea-...",
      "Compress": true
    }
  ]
}
```

→ Priority 순서 (위에서 아래). Default Behavior = lowest priority.

## 10. Invalidation (Cache Flush)

```bash
aws cloudfront create-invalidation --distribution-id $DIST_ID \
  --paths "/index.html" "/static/*" "/api/*"
```

| 비용 | 첫 1000 path/월 무료 / 추가 $0.005/path |

## 11. Stale Cache

```
Cache-Control: max-age=3600, stale-while-revalidate=86400
  → 1h 후 stale, 24h 동안 stale 응답 + background refresh
  
Cache-Control: max-age=3600, stale-if-error=86400
  → Origin error 시 24h 동안 stale 응답
```

## 12. Cache Statistics (CloudWatch)

```
Requests:           total
CacheHitRate:       cache hit %
BytesDownloaded:    DT
4xxErrorRate:       4xx
5xxErrorRate:       5xx
OriginLatency:      origin 응답 시간
TotalErrorRate
```

```bash
aws cloudwatch get-metric-statistics --namespace AWS/CloudFront \
  --metric-name CacheHitRate \
  --dimensions Name=DistributionId,Value=$DIST_ID Name=Region,Value=Global \
  --period 300 --statistics Average \
  --start-time $(date -u -v -1H +"%Y-%m-%dT%H:%M:%SZ") \
  --end-time $(date -u +"%Y-%m-%dT%H:%M:%SZ")
```

> CloudFront 의 metric = global (us-east-1).

## 13. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Cache Key 에 Authorization → user 별 cache (storage 폭주) | header 빼고 cache |
| API path 의 default 1d cache (오래된 응답) | CachingDisabled |
| Origin 의 Cache-Control 없음 → default TTL | 명시 |
| Invalidation 의 path 너무 많음 → cost ↑ | wildcard 또는 version 변경 |
| TTL 짧음 → cache hit ↓ | 적절한 TTL |
| SPA 의 index.html 강한 cache → 새 deploy 안 보임 | index.html 짧은 TTL, asset 긴 TTL + hashed name |

## 14. SPA 권장 Cache 패턴

```
index.html (entrypoint):
  Cache-Control: no-cache (always revalidate)
  TTL: 0
  
JS / CSS (hashed filename — e.g. app.abc123.js):
  Cache-Control: max-age=31536000, immutable
  TTL: 1y (영구)
  
Images / Fonts:
  Cache-Control: max-age=2592000
  TTL: 30d
  
API responses:
  CachingDisabled
```

## 15. 다음

- ▶ [[03-pricing-and-limits]]
- ▶ [[../02-features/01-https-and-oac]]
- ▶ [[../02-features/02-functions-and-lambda-edge]]

## 16. 영상 / 자료

| 자료 |
| --- |
| [Cache Policy](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/controlling-the-cache-key.html) |
| [Origin Request Policy](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/controlling-origin-requests.html) |
| [Response Headers Policy](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/modifying-response-headers.html) |

## 17. 참고

- [[../cloudfront]]
- [[01-distribution-and-origin]]
