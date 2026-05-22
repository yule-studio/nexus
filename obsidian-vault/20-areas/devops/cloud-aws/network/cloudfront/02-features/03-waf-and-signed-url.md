---
title: "CloudFront 02-3 — WAF + Signed URL / Cookie + Geo Restriction"
kind: knowledge
status: current
created_at: 2026-05-23T11:06:00+09:00
tags: [aws, cloudfront, features, waf, signed-url, signed-cookie, geo, shield]
home_hub: cloudfront
---

# CloudFront WAF + Signed URL + Geo Restriction

> WAF Web ACL associate + Signed URL/Cookie (paid content) + Geo restriction + Shield Standard 자동.

## 1. WAF Web ACL — CloudFront scope

```bash
# Web ACL — Scope = CLOUDFRONT (us-east-1 만)
aws wafv2 create-web-acl \
  --name bkurs-cf-waf \
  --scope CLOUDFRONT \
  --region us-east-1 \
  --default-action Allow={} \
  --rules '[
    { "Name":"CommonRules", "Priority":1,
      "Statement":{"ManagedRuleGroupStatement":{"VendorName":"AWS","Name":"AWSManagedRulesCommonRuleSet"}},
      "OverrideAction":{"None":{}},
      "VisibilityConfig":{"SampledRequestsEnabled":true,"CloudWatchMetricsEnabled":true,"MetricName":"common"} },
    { "Name":"RateLimit", "Priority":2,
      "Statement":{"RateBasedStatement":{"Limit":2000,"AggregateKeyType":"IP"}},
      "Action":{"Block":{}},
      "VisibilityConfig":{"SampledRequestsEnabled":true,"CloudWatchMetricsEnabled":true,"MetricName":"rate"} },
    { "Name":"BotControl", "Priority":3,
      "Statement":{"ManagedRuleGroupStatement":{"VendorName":"AWS","Name":"AWSManagedRulesBotControlRuleSet"}},
      "OverrideAction":{"None":{}},
      "VisibilityConfig":{"SampledRequestsEnabled":true,"CloudWatchMetricsEnabled":true,"MetricName":"bot"} }
  ]' \
  --visibility-config "SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=bkurs-cf"

# Distribution attach
aws cloudfront update-distribution --id $DIST_ID \
  --distribution-config "$(aws cloudfront get-distribution-config --id $DIST_ID --query 'DistributionConfig' | jq '.WebACLId = "arn:aws:wafv2:us-east-1:ACCT:global/webacl/bkurs-cf-waf/xxx"')" \
  --if-match $(aws cloudfront get-distribution-config --id $DIST_ID --query 'ETag' --output text)
```

> CloudFront 의 WAF = `--scope CLOUDFRONT --region us-east-1` 강제.

## 2. Geo Restriction (built-in)

```json
"Restrictions": {
  "GeoRestriction": {
    "RestrictionType": "whitelist",
    "Quantity": 3,
    "Items": ["KR", "US", "JP"]
  }
}
```

| `RestrictionType` | 의미 |
| --- | --- |
| `whitelist` | 명시된 country 만 access |
| `blacklist` | 명시된 country 차단 |
| `none` | 제한 없음 |

> 무료. CloudFront edge 에서 차단.

## 3. WAF vs Geo Restriction

| | Geo Restriction (built-in) | WAF GeoMatchStatement |
| --- | --- | --- |
| 비용 | 무료 | $1/rule/月 |
| Granularity | 모든 path | path / method / header combo |
| Action | block | block / count / captcha / challenge |
| Logging | ✗ | ✓ |

→ 단순 차단 = built-in. 정밀 제어 = WAF.

## 4. Signed URL — 임시 access

paid content / private file:
```python
# Python (CloudFront SDK)
from datetime import datetime, timedelta
import boto3
from botocore.signers import CloudFrontSigner
from cryptography.hazmat.primitives.serialization import load_pem_private_key

def rsa_signer(message):
    with open('private_key.pem', 'rb') as f:
        private_key = load_pem_private_key(f.read(), password=None)
    return private_key.sign(message, padding.PKCS1v15(), hashes.SHA1())

signer = CloudFrontSigner(KEY_PAIR_ID, rsa_signer)

# Canned Policy (단일 URL + expiration)
url = signer.generate_presigned_url(
    'https://bkurs.com/videos/premium.mp4',
    date_less_than=datetime.utcnow() + timedelta(hours=1)
)

# Custom Policy (IP range, multi-resource)
custom_policy = json.dumps({
    "Statement": [{
        "Resource": "https://bkurs.com/videos/*",
        "Condition": {
            "DateLessThan": {"AWS:EpochTime": int((datetime.utcnow() + timedelta(hours=1)).timestamp())},
            "IpAddress": {"AWS:SourceIp": "203.0.113.0/24"}
        }
    }]
})
url = signer.generate_presigned_url(
    'https://bkurs.com/videos/premium.mp4',
    policy=custom_policy
)
```

## 5. Signed URL — Distribution 설정

```bash
# Key Group (public key 들 묶음)
aws cloudfront create-public-key --public-key-config '{
  "CallerReference": "bkurs-key-1",
  "Name": "bkurs-public-key",
  "EncodedKey": "-----BEGIN PUBLIC KEY-----\n...\n-----END PUBLIC KEY-----"
}'

aws cloudfront create-key-group --key-group-config '{
  "Name": "bkurs-keygroup",
  "Items": ["{public_key_id}"]
}'

# Distribution 의 Behavior 에 TrustedKeyGroups 추가
"DefaultCacheBehavior": {
  ...,
  "TrustedKeyGroups": {
    "Enabled": true,
    "Quantity": 1,
    "Items": ["{key_group_id}"]
  }
}
```

## 6. Signed Cookie — 다중 file access

```python
# Cookie 발급
from botocore.signers import CloudFrontSigner

signer = CloudFrontSigner(KEY_PAIR_ID, rsa_signer)

# Custom Policy
policy = {
    "Statement": [{
        "Resource": "https://bkurs.com/private/*",
        "Condition": {
            "DateLessThan": {"AWS:EpochTime": int((datetime.utcnow() + timedelta(hours=1)).timestamp())}
        }
    }]
}

cookies = signer.generate_presigned_url(...)  # 또는 직접 cookie 생성
# {
#   "CloudFront-Policy": "...",
#   "CloudFront-Signature": "...",
#   "CloudFront-Key-Pair-Id": "..."
# }

# Response 에 Set-Cookie 추가
response.set_cookie('CloudFront-Policy', cookies['CloudFront-Policy'], domain='.bkurs.com', secure=True, httponly=True)
response.set_cookie('CloudFront-Signature', cookies['CloudFront-Signature'], domain='.bkurs.com', secure=True, httponly=True)
response.set_cookie('CloudFront-Key-Pair-Id', cookies['CloudFront-Key-Pair-Id'], domain='.bkurs.com', secure=True, httponly=True)
```

## 7. Signed URL vs Signed Cookie

| | Signed URL | Signed Cookie |
| --- | --- | --- |
| 사용처 | 단일 file 의 임시 link | 여러 file 의 사용자 session |
| 변경 | URL 바뀜 | URL 그대로 |
| Email 으로 공유 | ✓ | ✗ (cookie 별도) |
| Video streaming | ✓ (HLS playlist 의 segment) | ✓ |
| File download | ✓ | ✓ |

## 8. Shield Standard (자동)

```
Shield Standard:
- 무료, 자동 활성
- L3-L4 attack (SYN flood, UDP flood)
- CloudFront / Route 53 / ALB / GA 자동 보호
- 별도 설정 없음
```

## 9. Shield Advanced

```bash
aws shield subscribe-to-proactive-engagement \
  --emergency-contact-list EmailAddress=ops@bkurs.com,PhoneNumber=+82-10-...

# Distribution protection
aws shield create-protection --name bkurs-cf-protection \
  --resource-arn arn:aws:cloudfront::ACCT:distribution/{DIST_ID}
```

| | |
| --- | --- |
| 비용 | $3000/月 + DT |
| Coverage | L3-L7 + Application |
| SRT 지원 | 24/7 |
| Cost protection | AWS bill 환불 |

> 큰 traffic / critical app 만.

## 10. Field-Level Encryption

```bash
# Public Key 등록
aws cloudfront create-public-key --public-key-config '{
  "CallerReference": "field-encryption-key",
  "Name": "field-key",
  "EncodedKey": "-----BEGIN PUBLIC KEY-----\n...\n-----END PUBLIC KEY-----"
}'

# Field-Level Encryption Profile
aws cloudfront create-field-level-encryption-profile --field-level-encryption-profile-config '{
  "Name": "credit-card-profile",
  "CallerReference": "profile-1",
  "EncryptionEntities": {
    "Quantity": 1,
    "Items": [{
      "PublicKeyId": "{public_key_id}",
      "ProviderId": "credit-card-provider",
      "FieldPatterns": {
        "Quantity": 1,
        "Items": ["credit-card-number","ssn"]
      }
    }]
  }
}'

# Config
aws cloudfront create-field-level-encryption-config --field-level-encryption-config '{
  "CallerReference": "config-1",
  "QueryArgProfileConfig": {...},
  "ContentTypeProfileConfig": {
    "ForwardWhenContentTypeIsUnknown": true,
    "ContentTypeProfiles": {
      "Quantity": 1,
      "Items": [{
        "ContentType": "application/x-www-form-urlencoded",
        "Format": "URLEncoded",
        "ProfileId": "{profile_id}"
      }]
    }
  }
}'
```

→ HTTP form field 의 sensitive data (credit card, SSN) 가 CloudFront → Origin 으로 encrypt. Origin 의 specific service 만 decrypt.

## 11. Real-time Logs

```bash
# Kinesis Data Stream
aws kinesis create-stream --stream-name bkurs-cf-logs --shard-count 1

# Real-time log config
aws cloudfront create-realtime-log-config --realtime-log-config '{
  "EndPoints": [{
    "StreamType": "Kinesis",
    "KinesisStreamConfig": {
      "RoleARN": "arn:aws:iam::ACCT:role/cf-realtime-logs",
      "StreamARN": "arn:aws:kinesis:us-east-1:ACCT:stream/bkurs-cf-logs"
    }
  }],
  "Fields": ["timestamp","c-ip","sc-status","cs-method","cs-uri-stem","cs-host","cs-user-agent","x-edge-result-type"],
  "Name": "bkurs-realtime",
  "SamplingRate": 100
}'

# Distribution Cache Behavior 에 attach
"RealtimeLogConfigArn": "arn:..."
```

→ Real-time analytics / Fraud detection / DDoS detection.

## 12. Standard Log (S3)

```json
"Logging": {
  "Enabled": true,
  "IncludeCookies": false,
  "Bucket": "bkurs-cf-logs.s3.amazonaws.com",
  "Prefix": "logs/"
}
```

> S3 에 log file 저장. delay ~ 5분 ~ 1시간.

## 13. CloudWatch Alarm

```bash
aws cloudwatch put-metric-alarm --alarm-name cf-5xx-spike \
  --metric-name 5xxErrorRate --namespace AWS/CloudFront \
  --statistic Average --period 60 --evaluation-periods 5 \
  --threshold 1 --comparison-operator GreaterThanThreshold \
  --dimensions Name=DistributionId,Value=$DIST_ID Name=Region,Value=Global \
  --region us-east-1

aws cloudwatch put-metric-alarm --alarm-name cf-cache-hit-low \
  --metric-name CacheHitRate --namespace AWS/CloudFront \
  --statistic Average --period 300 --evaluation-periods 3 \
  --threshold 50 --comparison-operator LessThanThreshold \
  --dimensions Name=DistributionId,Value=$DIST_ID Name=Region,Value=Global \
  --region us-east-1
```

## 14. 권장 baseline — bkurs-be

```
[ ] WAF Web ACL (CLOUDFRONT scope, us-east-1)
[ ] AWS Managed Rules (Common / Known Bad / Amazon IP)
[ ] Rate-based rule (2000 req/5min/IP)
[ ] Geo restriction (필요 시 whitelist)
[ ] Signed URL/Cookie (paid content)
[ ] Shield Standard (자동)
[ ] Standard Log → S3 + Athena
[ ] CloudWatch alarm (5xx / Cache Hit)
[ ] CF Functions for SPA rewrite
[ ] Lambda@Edge for security headers (or Response Headers Policy)
```

## 15. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| WAF scope wrong (REGIONAL 대신 CLOUDFRONT) | us-east-1 + CLOUDFRONT |
| Signed URL 의 private key 분실 | KMS 활용 + rotation |
| Geo Restriction = 무료 인데 WAF 사용 | 단순 차단은 built-in |
| Real-time Logs cost 인지 못 함 | sampling rate 조정 |
| Shield Advanced 비용 ($3000/月) | high-value 만 |
| Trusted Key Groups 없이 Signed URL | enable |

## 16. 다음

- ▶ [[../practice/01-static-site-cdn]]
- ▶ [[../practice/02-api-acceleration-and-l-e]]
- ▶ [[../../route53/route53]]

## 17. 영상 / 자료

| 자료 |
| --- |
| [CloudFront WAF](https://docs.aws.amazon.com/waf/latest/developerguide/cloudfront-features.html) |
| [Signed URL/Cookie](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/PrivateContent.html) |
| [Geo Restriction](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/georestrictions.html) |

## 18. 참고

- [[../cloudfront]]
- [[01-https-and-oac]]
- [[02-functions-and-lambda-edge]]
- [[../../vpc/03-security/02-network-firewall-and-waf]]
