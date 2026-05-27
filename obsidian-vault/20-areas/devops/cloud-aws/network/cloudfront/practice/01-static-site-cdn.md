---
title: "CloudFront 실습 1 — Static Site + OAC + ACM + Route 53 + CF Functions"
kind: practice
status: current
created_at: 2026-05-23T11:10:00+09:00
tags: [aws, cloudfront, practice, oac, acm, route53, cf-functions, spa]
home_hub: cloudfront
---

# 실습 1 — Static Site + CloudFront

> 목표: S3 fully private + CloudFront OAC + ACM us-east-1 + Route 53 alias + CF Functions (SPA rewrite). 45분.

> 이 실습은 [[../../../storage/s3/practice/01-static-site-cloudfront]] 와 비슷. CloudFront 의 deep 부분 추가.

## 1. S3 + Content

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)
BUCKET=lab-static-$ACCT-$(date +%s)

aws s3 mb s3://$BUCKET
aws s3api put-public-access-block --bucket $BUCKET \
  --public-access-block-configuration "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# SPA structure
mkdir site
cat > site/index.html <<'EOF'
<!DOCTYPE html>
<html><head><title>Bkurs SPA</title></head><body>
<h1>Bkurs Static SPA</h1>
<div id="content">Loading...</div>
<script>
fetch('/api/info').then(r=>r.json()).then(d => document.getElementById('content').innerHTML = JSON.stringify(d));
</script>
</body></html>
EOF

cat > site/about/index.html <<'EOF'
<h1>About</h1>
EOF

cat > site/contact/index.html <<'EOF'
<h1>Contact</h1>
EOF

mkdir site/static
echo 'console.log("loaded")' > site/static/app.abc123.js
echo 'body { font-family: sans-serif; }' > site/static/main.def456.css

cat > site/404.html <<'EOF'
<h1>404 Not Found</h1><a href="/">Go home</a>
EOF

# Sync to S3 with cache-control
aws s3 sync site/ s3://$BUCKET/ \
  --cache-control "max-age=300" \
  --exclude "static/*"

aws s3 sync site/static/ s3://$BUCKET/static/ \
  --cache-control "max-age=31536000, immutable" \
  --metadata-directive REPLACE

aws s3 cp site/index.html s3://$BUCKET/index.html \
  --cache-control "no-cache" \
  --metadata-directive REPLACE
```

## 2. OAC

```bash
OAC_ID=$(aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "lab-static-oac",
    "Description": "OAC",
    "OriginAccessControlOriginType": "s3",
    "SigningBehavior": "always",
    "SigningProtocol": "sigv4"
  }' --query 'OriginAccessControl.Id' --output text)
```

## 3. ACM Certificate (us-east-1)

```bash
# 실제는 자신의 domain 사용
CERT_ARN=$(aws acm request-certificate \
  --domain-name lab.example.com \
  --validation-method DNS \
  --region us-east-1 \
  --query 'CertificateArn' --output text)

# Skip validation for lab (실제는 Route 53 의 CNAME 추가)
# 실습 — 임시 cert 없이 default CloudFront SSL 활용
```

## 4. CloudFront Function (SPA rewrite)

```bash
cat > cf-func.js <<'EOF'
function handler(event) {
    var request = event.request;
    var uri = request.uri;
    
    // /about → /about/index.html (no trailing slash + no extension)
    if (uri.endsWith('/')) {
        request.uri += 'index.html';
    } else if (!uri.includes('.')) {
        request.uri += '/index.html';
    }
    
    return request;
}
EOF

aws cloudfront create-function --name lab-spa-rewrite \
  --function-config '{"Comment":"SPA rewrite","Runtime":"cloudfront-js-2.0"}' \
  --function-code fileb://cf-func.js

FN_ETAG=$(aws cloudfront describe-function --name lab-spa-rewrite --query 'ETag' --output text)
aws cloudfront publish-function --name lab-spa-rewrite --if-match $FN_ETAG
```

## 5. Response Headers Policy

```bash
RHP_ID=$(aws cloudfront create-response-headers-policy --response-headers-policy-config '{
  "Name": "lab-security",
  "Comment": "Security headers",
  "SecurityHeadersConfig": {
    "StrictTransportSecurity": {
      "Override": true,
      "AccessControlMaxAgeSec": 63072000,
      "IncludeSubdomains": true,
      "Preload": true
    },
    "ContentTypeOptions": { "Override": true },
    "FrameOptions": { "Override": true, "FrameOption": "DENY" },
    "ReferrerPolicy": { "Override": true, "ReferrerPolicy": "strict-origin-when-cross-origin" }
  }
}' --query 'ResponseHeadersPolicy.Id' --output text)
```

## 6. Distribution

```bash
DIST_JSON=$(cat <<EOF
{
  "CallerReference": "lab-spa-$(date +%s)",
  "Comment": "lab SPA",
  "DefaultRootObject": "index.html",
  "Origins": {
    "Quantity": 1,
    "Items": [{
      "Id": "S3-$BUCKET",
      "DomainName": "$BUCKET.s3.ap-northeast-2.amazonaws.com",
      "OriginAccessControlId": "$OAC_ID",
      "S3OriginConfig": { "OriginAccessIdentity": "" }
    }]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "S3-$BUCKET",
    "ViewerProtocolPolicy": "redirect-to-https",
    "AllowedMethods": { "Quantity": 2, "Items": ["GET","HEAD"] },
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "ResponseHeadersPolicyId": "$RHP_ID",
    "Compress": true,
    "FunctionAssociations": {
      "Quantity": 1,
      "Items": [{
        "FunctionARN": "arn:aws:cloudfront::$ACCT:function/lab-spa-rewrite",
        "EventType": "viewer-request"
      }]
    }
  },
  "CustomErrorResponses": {
    "Quantity": 1,
    "Items": [{
      "ErrorCode": 403,
      "ResponsePagePath": "/index.html",
      "ResponseCode": "200",
      "ErrorCachingMinTTL": 60
    }]
  },
  "Enabled": true,
  "PriceClass": "PriceClass_200",
  "HttpVersion": "http2and3",
  "IsIPV6Enabled": true
}
EOF
)

DIST_ID=$(aws cloudfront create-distribution --distribution-config "$DIST_JSON" \
  --query 'Distribution.Id' --output text)

DIST_DOMAIN=$(aws cloudfront get-distribution --id $DIST_ID --query 'Distribution.DomainName' --output text)

echo "Distribution: $DIST_ID"
echo "Domain: $DIST_DOMAIN"
```

> SPA 의 client routing = `CustomErrorResponses` 의 403 → `/index.html` 로.

## 7. S3 Bucket Policy

```bash
aws s3api put-bucket-policy --bucket $BUCKET --policy "$(cat <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontOAC",
      "Effect": "Allow",
      "Principal": { "Service": "cloudfront.amazonaws.com" },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::$BUCKET/*",
      "Condition": {
        "StringEquals": { "AWS:SourceArn": "arn:aws:cloudfront::$ACCT:distribution/$DIST_ID" }
      }
    }
  ]
}
EOF
)"
```

## 8. Distribution 배포 대기 + 검증

```bash
echo "Waiting for distribution deployment..."
aws cloudfront wait distribution-deployed --id $DIST_ID

curl -L https://$DIST_DOMAIN/
# <h1>Bkurs Static SPA</h1>...

curl -L https://$DIST_DOMAIN/about
# <h1>About</h1>  (CF Function rewrite)

curl -L https://$DIST_DOMAIN/unknown
# <h1>Bkurs Static SPA</h1>  (SPA fallback via CustomErrorResponses)

curl -I https://$DIST_DOMAIN/static/app.abc123.js
# Cache-Control: max-age=31536000, immutable
# x-cache: Hit from cloudfront

# Security headers (Response Headers Policy)
curl -I https://$DIST_DOMAIN/ | grep -iE "(strict|content-type-options|frame-options|referrer)"
```

## 9. Cache Hit Rate 측정

```bash
# CloudWatch metric (us-east-1)
aws cloudwatch get-metric-statistics --namespace AWS/CloudFront \
  --metric-name CacheHitRate \
  --dimensions Name=DistributionId,Value=$DIST_ID Name=Region,Value=Global \
  --period 300 --statistics Average \
  --start-time $(date -u -v -1H +"%Y-%m-%dT%H:%M:%SZ") \
  --end-time $(date -u +"%Y-%m-%dT%H:%M:%SZ") \
  --region us-east-1
```

## 10. Invalidation (cache flush)

```bash
# index.html 만 (CSS / JS 는 hashed name 이라 그대로)
aws cloudfront create-invalidation --distribution-id $DIST_ID \
  --paths "/index.html"

# Full
aws cloudfront create-invalidation --distribution-id $DIST_ID \
  --paths "/*"
```

## 11. Route 53 alias (옵션, 실제 domain 필요)

```bash
HZ=$(aws route53 list-hosted-zones-by-name --dns-name example.com \
  --query 'HostedZones[0].Id' --output text | sed 's|/hostedzone/||')

aws route53 change-resource-record-sets --hosted-zone-id $HZ \
  --change-batch "{
    \"Changes\": [{
      \"Action\": \"UPSERT\",
      \"ResourceRecordSet\": {
        \"Name\": \"lab.example.com.\",
        \"Type\": \"A\",
        \"AliasTarget\": {
          \"HostedZoneId\": \"Z2FDTNDATAQYW2\",
          \"DNSName\": \"$DIST_DOMAIN\",
          \"EvaluateTargetHealth\": false
        }
      }
    }]
  }"
```

> CloudFront 의 hosted zone ID = `Z2FDTNDATAQYW2` (고정).

## 12. Clean up

```bash
# Disable distribution
ETAG=$(aws cloudfront get-distribution-config --id $DIST_ID --query 'ETag' --output text)
CONFIG=$(aws cloudfront get-distribution-config --id $DIST_ID --query 'DistributionConfig' | jq '.Enabled = false')

aws cloudfront update-distribution --id $DIST_ID \
  --distribution-config "$CONFIG" --if-match $ETAG

aws cloudfront wait distribution-deployed --id $DIST_ID

# Delete
ETAG2=$(aws cloudfront get-distribution-config --id $DIST_ID --query 'ETag' --output text)
aws cloudfront delete-distribution --id $DIST_ID --if-match $ETAG2

# OAC
OAC_ETAG=$(aws cloudfront get-origin-access-control --id $OAC_ID --query 'ETag' --output text)
aws cloudfront delete-origin-access-control --id $OAC_ID --if-match $OAC_ETAG

# Response Headers Policy
aws cloudfront delete-response-headers-policy --id $RHP_ID \
  --if-match $(aws cloudfront get-response-headers-policy --id $RHP_ID --query 'ETag' --output text)

# CF Function
aws cloudfront delete-function --name lab-spa-rewrite \
  --if-match $(aws cloudfront describe-function --name lab-spa-rewrite --query 'ETag' --output text)

# S3
aws s3 rm s3://$BUCKET --recursive
aws s3 rb s3://$BUCKET
```

## 13. 학습 포인트

- S3 fully private (BPA) + OAC + Bucket Policy SourceArn
- CloudFront Functions 의 SPA rewrite
- CustomErrorResponses 의 SPA fallback (403 → /index.html)
- Cache 의 differential strategy (index.html no-cache / static immutable)
- Response Headers Policy 의 security headers
- ACM cert us-east-1 강제
- Route 53 alias (`Z2FDTNDATAQYW2`)
- Distribution disable + delete 의 2-step

## 14. 다음

- ▶ [[02-api-acceleration-and-l-e]]
- ▶ [[../../route53/route53]]

## 15. 참고

- [[../cloudfront]]
- [[../01-fundamentals/01-distribution-and-origin]]
- [[../02-features/01-https-and-oac]]
- [[../02-features/02-functions-and-lambda-edge]]
- [[../../../storage/s3/practice/01-static-site-cloudfront]]
