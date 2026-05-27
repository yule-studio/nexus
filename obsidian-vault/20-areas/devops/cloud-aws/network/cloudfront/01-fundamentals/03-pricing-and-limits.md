---
title: "CloudFront 01-3 — Pricing + Limits"
kind: knowledge
status: current
created_at: 2026-05-23T11:00:00+09:00
tags: [aws, cloudfront, fundamentals, pricing, limits]
home_hub: cloudfront
---

# CloudFront Pricing + Limits

> Per-region DT + Request + Function/L@E + Optional features (Shield Advanced, Origin Shield).

## 1. 가격 구성

```
1. Data Transfer Out (region 별 tiered)
2. Request (HTTPS / HTTP)
3. CloudFront Functions / Lambda@Edge
4. Origin Shield (optional)
5. Field-Level Encryption (optional)
6. Real-time Logs (optional)
```

## 2. Data Transfer Out — Region Tier

| Region | First 10TB / 월 |
| --- | --- |
| US, EU, IL | $0.085/GB |
| Canada | $0.085/GB |
| Mexico | $0.110/GB |
| **Asia (Seoul / Tokyo)** | **$0.140/GB** |
| Japan | $0.114/GB |
| Australia | $0.114/GB |
| India | $0.109/GB |
| South America | $0.110/GB |
| Africa | $0.110/GB |

→ Tiered (10TB / 40TB / 100TB / 350TB / 524TB) — volume 클수록 단가 ↓.

## 3. Free Tier

```
Free / 월:
- 1 TB DT outbound
- 10M HTTP requests
- 2M CloudFront Functions
- 무료 SSL cert (ACM)
- 무제한 distribution
```

## 4. Request Pricing

| | $/10K request |
| --- | --- |
| HTTPS (Asia) | $0.0120 |
| HTTP (Asia) | $0.0090 |
| HTTPS (US) | $0.0100 |
| HTTP (US) | $0.0075 |

## 5. CloudFront Functions vs Lambda@Edge

| | CloudFront Functions | Lambda@Edge |
| --- | --- | --- |
| 언어 | JavaScript (ES 2020+) | Node.js / Python |
| 실행 | < 1ms | < 5초 |
| 메모리 | 2 MB | 128 MB - 10 GB |
| 비용 | $0.10/M req | $0.60/M req + $0.0000125/GB-s |
| 위치 | Edge (모든 POP) | Regional Edge Cache (13 region) |
| Event | viewer-request / viewer-response | + origin-request / origin-response |
| Network call | ✗ | ✓ |
| 용도 | header 조작 / redirect / URL rewrite | 복잡 logic / external API |

## 6. Origin Shield Pricing

```
$0.0075 / 10K request
+ DT (cross-region)
```

활용:
- Origin 부하 큰 경우 (cache miss 줄임)
- Live event
- 비싼 origin (EC2 / on-prem)

## 7. Field-Level Encryption

```
$0 (요청 자체)
+ Public Key 관리 비용
+ Lambda@Edge (decrypt 시)
```

활용: PII (credit card / SSN) 의 client → backend 전달 시 encrypt.

## 8. Real-time Logs

```
$0.010 / 1M log records
+ Kinesis Data Stream cost
```

활용:
- Real-time analytics
- Fraud detection
- DDoS detection

## 9. 비용 시뮬레이션 — bkurs-be

```
1 TB outbound (Asia tier 1):
  1024 GB × $0.140 = $143/월

20M HTTPS request:
  20 × $0.0120 = $0.24/월

CloudFront Functions:
  5M × $0.10/M = $0.50/월

Total: ~$144/월

vs S3 direct (without CF):
  1 TB × $0.114 = $117/월 (cheaper!)
  
→ Origin DT vs CF DT 비교 필요. Cache hit 가 비용 절감의 key.
```

### Cache Hit Effect

```
Origin DT (S3 → CF): free
CF → Internet:        $0.140/GB

만약 cache hit rate 80%:
  Origin DT = 20% only (cache miss)
  → S3 cost = 20% × original

만약 cache hit rate 0% (no cache):
  Origin DT 비용은 같음
  + CF 의 추가 cost
  → CF 가 더 비싸짐
```

→ **Cache hit > 50% 일 때 CloudFront 가 cost saving**.

## 10. Cost 절감 전략

```
1. Cache hit rate ↑ (정적 asset 의 긴 TTL)
2. Origin Shield (cache miss 줄임)
3. Compress (Gzip / Brotli)
4. Modern image format (WebP / AVIF)
5. Price Class 제한 (PriceClass_100 = US/EU 만, 30-40% 절감)
6. HTTPS only (HTTP 보다 약간 비싸지만 보안)
7. CloudFront Functions (Lambda@Edge 보다 6x 싸다)
8. Compression (Gzip / Brotli)
```

## 11. Hard Limits

| | |
| --- | --- |
| Distributions / account | 200 (soft, max 5000) |
| Aliases / distribution | 100 |
| Origins / distribution | 25 |
| Origin Groups / distribution | 10 |
| Cache Behaviors / distribution | 25 |
| Cache Policies / account | 50 |
| Origin Request Policies / account | 50 |
| Response Headers Policies / account | 50 |
| Function (CloudFront Functions) / distribution | 2 (viewer-request, viewer-response) |
| Lambda@Edge / distribution | 4 (viewer-req/res + origin-req/res) |
| Field-level encryption configs | 10 |
| Real-time log configs / account | 10 |
| Invalidation paths / request | 3000 |
| Invalidation paths free / 월 | 1000 |
| Custom error pages / distribution | 25 |
| TLS Cert per distribution | 1 (default) + 25 SNI |

## 12. Service Quotas 증액

```bash
aws service-quotas list-service-quotas --service-code cloudfront \
  --query "Quotas[?Adjustable==\`true\`].[QuotaName,Value,QuotaCode]" --output table

aws service-quotas request-service-quota-increase \
  --service-code cloudfront \
  --quota-code L-24B04930 \
  --desired-value 500
```

## 13. CloudWatch metric (Global)

```bash
aws cloudwatch get-metric-statistics --namespace AWS/CloudFront \
  --metric-name BytesDownloaded \
  --dimensions Name=DistributionId,Value=$DIST_ID Name=Region,Value=Global \
  --period 3600 --statistics Sum \
  --start-time $(date -u -v -1d +"%Y-%m-%dT%H:%M:%SZ") \
  --end-time $(date -u +"%Y-%m-%dT%H:%M:%SZ")
```

| Metric | 의미 |
| --- | --- |
| `Requests` | total |
| `BytesDownloaded` | DT out |
| `BytesUploaded` | DT in |
| `CacheHitRate` | hit % |
| `4xxErrorRate` / `5xxErrorRate` | error |
| `OriginLatency` | origin 응답 시간 |
| `TotalErrorRate` | 모든 error |
| `FunctionInvocations` | CF Functions |
| `FunctionExecutionErrors` | |

> CloudFront metric region = **us-east-1** (CloudWatch).

## 14. Cost Allocation Tag

```bash
aws cloudfront tag-resource --resource $DIST_ARN \
  --tags 'Items=[{Key=Environment,Value=prod},{Key=Service,Value=bkurs},{Key=CostCenter,Value=engineering}]'
```

## 15. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| Cache hit rate ↓ → 비용 증가 | TTL + Cache Key 최적화 |
| Price Class All (모든 region) | _100 / _200 (절감) |
| Lambda@Edge 매번 호출 | CF Functions 또는 origin cache |
| Compression 안 함 | Gzip / Brotli enable |
| Invalidation 의 path 너무 많음 | wildcard or version |
| Real-time Logs 비용 인지 못 함 | 필요할 때만 |

## 16. 다음

- ▶ [[../02-features/01-https-and-oac]]
- ▶ [[../02-features/02-functions-and-lambda-edge]]
- ▶ [[../02-features/03-waf-and-signed-url]]

## 17. 영상 / 자료

| 자료 |
| --- |
| [CloudFront Pricing](https://aws.amazon.com/cloudfront/pricing/) |
| [Free Tier](https://aws.amazon.com/cloudfront/) |
| [Quotas](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-limits.html) |

## 18. 참고

- [[../cloudfront]]
- [[01-distribution-and-origin]]
- [[02-cache-behavior-and-policies]]
