---
title: "S3 실습 1 — Static Site + CloudFront + Route 53 + OAC"
kind: practice
status: current
created_at: 2026-05-22T19:02:00+09:00
tags: [aws, s3, practice, cloudfront, static-site, oac, route53]
home_hub: s3
---

# 실습 1 — Static Site + CloudFront + OAC

> 목표: S3 static site + CloudFront + Route 53 도메인 + OAC (Origin Access Control). 30분.

## 1. S3 bucket (private)

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)
BUCKET=lab-static-$ACCT-$(date +%s)

aws s3 mb s3://$BUCKET --region ap-northeast-2

# Block Public Access (full)
aws s3api put-public-access-block --bucket $BUCKET \
  --public-access-block-configuration "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Default encryption SSE-S3
# 2023+ 자동
```

## 2. Static content

```bash
mkdir site
cat > site/index.html <<'EOF'
<!DOCTYPE html>
<html>
<head><title>Bkurs</title></head>
<body>
  <h1>Hello from Bkurs Static!</h1>
  <p>Served via CloudFront + S3</p>
  <script src="/app.js"></script>
</body>
</html>
EOF

cat > site/404.html <<'EOF'
<h1>404 Not Found</h1>
EOF

cat > site/app.js <<'EOF'
console.log("Bkurs static");
EOF

aws s3 sync site/ s3://$BUCKET/ --cache-control "max-age=3600"
```

## 3. CloudFront Origin Access Control (OAC)

```bash
OAC_ID=$(aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "lab-static-oac",
    "Description": "OAC for lab-static",
    "OriginAccessControlOriginType": "s3",
    "SigningBehavior": "always",
    "SigningProtocol": "sigv4"
  }' --query 'OriginAccessControl.Id' --output text)

echo "OAC_ID=$OAC_ID"
```

## 4. CloudFront Distribution

```bash
DIST_JSON=$(cat <<EOF
{
  "CallerReference": "lab-static-$(date +%s)",
  "Comment": "lab static",
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
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6"
  },
  "CustomErrorResponses": {
    "Quantity": 1,
    "Items": [{
      "ErrorCode": 404,
      "ResponsePagePath": "/404.html",
      "ResponseCode": "404",
      "ErrorCachingMinTTL": 60
    }]
  },
  "Enabled": true,
  "PriceClass": "PriceClass_200"
}
EOF
)

DIST_ID=$(aws cloudfront create-distribution --distribution-config "$DIST_JSON" \
  --query 'Distribution.Id' --output text)
DIST_DOMAIN=$(aws cloudfront get-distribution --id $DIST_ID --query 'Distribution.DomainName' --output text)

echo "DIST_ID=$DIST_ID"
echo "DIST_DOMAIN=$DIST_DOMAIN"
```

## 5. Bucket Policy — CloudFront OAC 만 접근

```bash
aws s3api put-bucket-policy --bucket $BUCKET --policy "$(cat <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFront",
      "Effect": "Allow",
      "Principal": { "Service": "cloudfront.amazonaws.com" },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::$BUCKET/*",
      "Condition": {
        "StringEquals": { "AWS:SourceArn": "arn:aws:cloudfront::$ACCT:distribution/$DIST_ID" }
      }
    },
    {
      "Sid": "DenyHTTP",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": ["arn:aws:s3:::$BUCKET/*", "arn:aws:s3:::$BUCKET"],
      "Condition": { "Bool": { "aws:SecureTransport": "false" } }
    }
  ]
}
EOF
)"
```

## 6. 검증 (CloudFront 배포 대기 ~10분)

```bash
echo "Waiting for distribution deployment..."
aws cloudfront wait distribution-deployed --id $DIST_ID

curl -L https://$DIST_DOMAIN/
# > <h1>Hello from Bkurs Static!</h1>
```

## 7. ACM Certificate (custom domain, us-east-1 만)

```bash
# CloudFront 의 ACM 은 반드시 us-east-1
CERT_ARN=$(aws acm request-certificate --domain-name "static.bkurs.com" \
  --validation-method DNS --region us-east-1 \
  --query 'CertificateArn' --output text)

# DNS validation record 추가 (Route 53)
aws acm describe-certificate --certificate-arn $CERT_ARN --region us-east-1 \
  --query 'Certificate.DomainValidationOptions[0].ResourceRecord'
# Name / Value 를 Route 53 에 CNAME 추가

aws acm wait certificate-validated --certificate-arn $CERT_ARN --region us-east-1
```

## 8. CloudFront 에 Custom Domain attach

```bash
# Distribution config 업데이트
aws cloudfront get-distribution-config --id $DIST_ID \
  --query 'DistributionConfig' > dist.json

# Aliases + ViewerCertificate 추가
jq --arg cert "$CERT_ARN" \
  '.Aliases = {"Quantity":1,"Items":["static.bkurs.com"]} | .ViewerCertificate = {"ACMCertificateArn":$cert,"SSLSupportMethod":"sni-only","MinimumProtocolVersion":"TLSv1.2_2021","Certificate":$cert,"CertificateSource":"acm"}' \
  dist.json > dist-updated.json

ETAG=$(aws cloudfront get-distribution-config --id $DIST_ID --query 'ETag' --output text)
aws cloudfront update-distribution --id $DIST_ID \
  --distribution-config file://dist-updated.json --if-match $ETAG
```

## 9. Route 53 A record (alias)

```bash
HOSTED_ZONE=$(aws route53 list-hosted-zones-by-name --dns-name bkurs.com \
  --query 'HostedZones[0].Id' --output text | sed 's|/hostedzone/||')

aws route53 change-resource-record-sets --hosted-zone-id $HOSTED_ZONE \
  --change-batch "{
    \"Changes\": [{
      \"Action\": \"UPSERT\",
      \"ResourceRecordSet\": {
        \"Name\": \"static.bkurs.com.\",
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

> `Z2FDTNDATAQYW2` = CloudFront 의 hosted zone ID (고정).

## 10. Invalidation (cache flush)

```bash
# 새 version 배포 후
aws s3 sync site-v2/ s3://$BUCKET/ --cache-control "max-age=3600"

aws cloudfront create-invalidation --distribution-id $DIST_ID \
  --paths "/*"
```

| 비용 |
| --- |
| 첫 1000 path/월 무료 |
| 추가 $0.005/path |

## 11. CloudFront Function (edge logic)

```bash
cat > cf-func.js <<'EOF'
function handler(event) {
  var request = event.request;
  if (!request.uri.includes('.')) {
    request.uri = request.uri + '/index.html';
  }
  return request;
}
EOF

FUNC_ETAG=$(aws cloudfront create-function --name lab-rewrite \
  --function-config '{"Comment":"rewrite","Runtime":"cloudfront-js-2.0"}' \
  --function-code fileb://cf-func.js \
  --query 'ETag' --output text)

aws cloudfront publish-function --name lab-rewrite --if-match $FUNC_ETAG
```

→ /about → /about/index.html 자동 rewrite.

## 12. Clean up

```bash
aws cloudfront update-distribution --id $DIST_ID \
  --distribution-config "$(aws cloudfront get-distribution-config --id $DIST_ID --query 'DistributionConfig' | jq '.Enabled = false')" \
  --if-match $(aws cloudfront get-distribution-config --id $DIST_ID --query 'ETag' --output text)
aws cloudfront wait distribution-deployed --id $DIST_ID
aws cloudfront delete-distribution --id $DIST_ID \
  --if-match $(aws cloudfront get-distribution-config --id $DIST_ID --query 'ETag' --output text)

aws cloudfront delete-origin-access-control --id $OAC_ID \
  --if-match $(aws cloudfront get-origin-access-control --id $OAC_ID --query 'ETag' --output text)

aws cloudfront delete-function --name lab-rewrite --if-match $FUNC_ETAG

aws s3 rm s3://$BUCKET/ --recursive
aws s3 rb s3://$BUCKET

# ACM cert
aws acm delete-certificate --certificate-arn $CERT_ARN --region us-east-1
```

## 13. 학습 포인트

- S3 fully private (BPA) + CloudFront OAC
- Bucket Policy 의 CloudFront SourceArn condition
- HTTPS 강제 (SecureTransport)
- ACM certificate us-east-1 만 (CloudFront)
- Route 53 alias (Z2FDTNDATAQYW2)
- CloudFront Function (edge rewrite)
- Cache invalidation

## 14. 다음

- ▶ [[02-presigned-upload-pipeline]]
- ▶ [[03-lifecycle-glacier-deep-archive]]
- ▶ [[../../../network/cloudfront/cloudfront]]

## 15. 참고

- [[../s3]]
- [[../02-access/02-block-public-access-and-vpc-endpoint]]
