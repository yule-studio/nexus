---
title: "AWS CloudFront — CDN"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:10:00+09:00
tags:
  - aws
  - network
  - cdn
  - cloudfront
---

# AWS CloudFront

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | CloudFront 개념 + 사용 |

**[[network|↑ Network]]** · **[[cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

AWS 의 **CDN** — 전 세계 400+ edge POP. 응답 latency ↓ + origin 부하 ↓ + DDoS 1차 방어.

---

## 2. 왜

- **글로벌 사용자** = 한국 origin 까지 100-200 ms. CDN edge 캐시 = 수십 ms.
- **origin 부하 ↓** — 캐시 hit 이면 origin 안 옴
- **TLS termination at edge** — 빠른 handshake
- **DDoS / WAF 1차 방어**
- **data transfer 가격 ↓** — origin → 인터넷 $0.09 vs CloudFront $0.085

대안:
- Cloudflare / Fastly — 더 풍부한 edge logic
- Akamai — 엔터프라이즈

---

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Distribution** | CDN 인스턴스 (DNS: `d111.cloudfront.net`) |
| **Origin** | back-end (S3 / ALB / EC2 / API GW / 외부) |
| **Behavior** | path 패턴 → 어떤 origin / TTL / cache key |
| **Cache Policy** | cache 키 (header / cookie / query 포함 여부) |
| **Origin Request Policy** | origin 에 전달할 header |
| **Function / Lambda@Edge** | edge 의 작은 logic |
| **WAF** | 통합 가능 |
| **OAC** | Origin Access Control (S3 private) |

---

## 4. 설치 / 사용

### 4.1 S3 + CloudFront (정적 사이트)

```hcl
resource "aws_s3_bucket" "site" {
  bucket = "my-site-static"
}

resource "aws_s3_bucket_public_access_block" "site" {
  bucket                  = aws_s3_bucket.site.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_cloudfront_origin_access_control" "site" {
  name                              = "my-site-oac"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

resource "aws_cloudfront_distribution" "site" {
  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"
  aliases             = ["www.example.com"]
  price_class         = "PriceClass_200"          # 사용 region 제한 (비용 ↓)

  origin {
    domain_name              = aws_s3_bucket.site.bucket_regional_domain_name
    origin_id                = "s3-site"
    origin_access_control_id = aws_cloudfront_origin_access_control.site.id
  }

  default_cache_behavior {
    target_origin_id       = "s3-site"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    compress               = true

    cache_policy_id        = "658327ea-f89d-4fab-a63d-7e88639e58f6"   # Managed-CachingOptimized
  }

  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.cert.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  restrictions {
    geo_restriction { restriction_type = "none" }
  }
}

# S3 bucket policy — OAC 만 read 허용
data "aws_iam_policy_document" "s3_oac" {
  statement {
    actions   = ["s3:GetObject"]
    resources = ["${aws_s3_bucket.site.arn}/*"]
    principals {
      type        = "Service"
      identifiers = ["cloudfront.amazonaws.com"]
    }
    condition {
      test     = "StringEquals"
      variable = "AWS:SourceArn"
      values   = [aws_cloudfront_distribution.site.arn]
    }
  }
}

resource "aws_s3_bucket_policy" "site" {
  bucket = aws_s3_bucket.site.id
  policy = data.aws_iam_policy_document.s3_oac.json
}
```

### 4.2 ALB origin (동적)

같은 distribution 안에서 path 별로:
```
/static/*  → S3
/api/*     → ALB (no cache)
/          → S3 index.html
```

```hcl
ordered_cache_behavior {
  path_pattern           = "/api/*"
  target_origin_id       = "alb-api"
  viewer_protocol_policy = "redirect-to-https"
  allowed_methods        = ["GET","HEAD","OPTIONS","POST","PUT","DELETE","PATCH"]
  cached_methods         = ["GET","HEAD"]

  cache_policy_id        = "4135ea2d-6df8-44a3-9df3-4b5a84be39ad"     # Managed-CachingDisabled
  origin_request_policy_id = "216adef6-5c7f-47e4-b989-5492eafa07d3"    # Managed-AllViewer
}
```

---

## 5. Managed Cache Policy

| Policy | 의도 |
| --- | --- |
| `Managed-CachingOptimized` | 정적 — query 무시 |
| `Managed-CachingDisabled` | 동적 — cache 안 함 |
| `Managed-CachingOptimizedForUncompressedObjects` | 이미 압축된 파일 |
| `Managed-Elemental-MediaPackage` | 비디오 |

자체 정의도 가능. cache key 에 어떤 header / cookie / query 포함.

---

## 6. invalidation

```bash
aws cloudfront create-invalidation \
  --distribution-id E2QWRUHAPOMQZL \
  --paths "/index.html" "/css/*"
```

→ deploy 후 cache 강제 무효화. **첫 1000 / 월 무료**, 이후 $0.005/path.

---

## 7. CloudFront Functions / Lambda@Edge

### 7.1 CloudFront Functions (μ초 latency, JavaScript)

```javascript
function handler(event) {
    var request = event.request;
    // /index → /index.html
    if (request.uri.endsWith('/')) {
        request.uri += 'index.html';
    }
    return request;
}
```

- Viewer Request / Viewer Response 만
- 1 ms 이하
- $0.10/M 호출

### 7.2 Lambda@Edge

- Viewer / Origin Request / Response
- Node.js / Python
- 5-30 초 timeout
- 무거운 logic / SSR
- 비쌈

---

## 8. 보안

- **HTTPS only** (`viewer_protocol_policy = "redirect-to-https"`)
- **OAC** — S3 origin private 보호
- **WAF** — distribution 에 직접 attach
- **Signed URL / Signed Cookie** — 시한 access
- **Geo restriction** — country block
- **Field-Level Encryption** — POST data 일부 암호화

---

## 9. 비용 (Seoul / Asia)

```
Data Transfer Out (Asia)   $0.114 / GB
HTTP Request               $0.012 / 10K
HTTPS Request              $0.014 / 10K
Invalidation 1000/월 무료, 이후 $0.005/path
CloudFront Functions       $0.10 / 1M invocation
Lambda@Edge                $0.60 / 1M + GB·s
```

→ origin → 인터넷 직보다 항상 더 저렴.

---

## 10. 사용 시나리오

- SPA 정적 (React / Vue / Next static export)
- 이미지 / 동영상 / PDF
- API + 정적 mix (path 별)
- 글로벌 사용자
- DDoS 1차 방어 (Shield Standard 자동)
- SSR + edge cache

부적합:
- 매번 다른 응답 (auth + 동적) — origin 직접 또는 ALB
- 매우 작은 사용량 (Lightsail / 직접)

---

## 11. 함정

### 11.1 cache key 의 query / cookie
잘못된 cache key = hit rate ↓ 또는 사용자 별 응답 mix. 명시.

### 11.2 invalidation 남용
비싸짐. versioned URL (`/static/v1.2.3/...`) 권장.

### 11.3 origin response 헤더 무시
`Cache-Control: no-cache` 가 무시되는 옵션 — Managed policy 사용 시 origin 헤더 override 가능.

### 11.4 ACM region
CloudFront 인증서 = **us-east-1 (Virginia)** 만. ALB 는 자기 region.

### 11.5 Lambda@Edge 디버그
Cloudwatch logs 가 edge 의 region 마다. 추적 어려움.

### 11.6 deploy 후 즉시 갱신 X
distribution 변경 = 5-15 분 propagate.

### 11.7 cors
ALB / S3 의 CORS 설정 + Cache 의 Vary header.

---

## 12. 학습 자료

- AWS CloudFront docs
- **CloudFront Workshop**
- **Best Practices for CloudFront** AWS whitepaper

---

## 13. 관련

- [[network]] — Network hub
- [[alb]] — 동적 origin
- [[s3]] — 정적 origin
- [[route53]] — DNS
- [[security]] — WAF
