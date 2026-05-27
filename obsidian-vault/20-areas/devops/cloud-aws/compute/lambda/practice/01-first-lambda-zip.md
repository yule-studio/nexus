---
title: "Lambda 실습 1 — Python zip + S3 trigger (image resize)"
kind: practice
status: current
created_at: 2026-05-22T17:24:00+09:00
tags: [aws, lambda, practice, python, s3]
home_hub: lambda
---

# 실습 1 — Python zip + S3 trigger

> 목표: S3 upload → Lambda → resize → 다른 prefix 저장. 20분.

## 사전 준비

- AWS CLI v2 + region `ap-northeast-2`
- Python 3.12

## 1. S3 bucket

```bash
ACCT=$(aws sts get-caller-identity --query Account --output text)
BUCKET=lab-img-$ACCT-$(date +%s)

aws s3 mb s3://$BUCKET
aws s3api put-bucket-encryption --bucket $BUCKET \
  --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
```

## 2. IAM Role

```bash
cat > trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "lambda.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role --role-name LabResizeRole \
  --assume-role-policy-document file://trust.json

cat > policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["logs:CreateLogGroup","logs:CreateLogStream","logs:PutLogEvents"],
      "Resource": "arn:aws:logs:ap-northeast-2:$ACCT:log-group:/aws/lambda/lab-resize:*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::$BUCKET/raw/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::$BUCKET/thumb/*"
    }
  ]
}
EOF

aws iam put-role-policy --role-name LabResizeRole \
  --policy-name LabResizePolicy --policy-document file://policy.json
```

## 3. Code

```python
# app.py
import boto3
import io
import os
import urllib.parse
from PIL import Image

s3 = boto3.client("s3")
BUCKET = os.environ["BUCKET"]
SIZE = int(os.environ.get("THUMB_SIZE", "200"))

def lambda_handler(event, context):
    for record in event["Records"]:
        key = urllib.parse.unquote_plus(record["s3"]["object"]["key"])
        if not key.startswith("raw/"):
            continue

        # 1. download
        obj = s3.get_object(Bucket=BUCKET, Key=key)
        img = Image.open(obj["Body"])

        # 2. thumbnail
        img.thumbnail((SIZE, SIZE))

        # 3. upload thumb/
        buf = io.BytesIO()
        img.save(buf, format=img.format or "JPEG")
        buf.seek(0)
        new_key = key.replace("raw/", "thumb/", 1)
        s3.put_object(Bucket=BUCKET, Key=new_key, Body=buf.getvalue(),
                      ContentType=f"image/{(img.format or 'jpeg').lower()}")

        print(f"resized: {key} → {new_key}")

    return {"statusCode": 200}
```

```bash
mkdir -p package
pip install -t package Pillow

cp app.py package/
cd package
zip -r ../function.zip .
cd ..
```

## 4. Lambda 생성

```bash
aws lambda create-function --function-name lab-resize \
  --runtime python3.12 --architectures arm64 \
  --handler app.lambda_handler \
  --role arn:aws:iam::$ACCT:role/LabResizeRole \
  --zip-file fileb://function.zip \
  --memory-size 512 --timeout 30 \
  --environment "Variables={BUCKET=$BUCKET,THUMB_SIZE=200}" \
  --tracing-config Mode=Active

aws logs put-retention-policy --log-group-name /aws/lambda/lab-resize --retention-in-days 7
```

> arm64 Pillow 가 호환 — `pip install -t package Pillow` 가 정확히 arm 으로 빌드돼야 한다.

빌드 시 platform 명시:
```bash
pip install --platform manylinux2014_aarch64 --only-binary=:all: \
  --target package Pillow
```

## 5. S3 → Lambda trigger

```bash
aws lambda add-permission --function-name lab-resize \
  --statement-id s3-trigger \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::$BUCKET \
  --source-account $ACCT

aws s3api put-bucket-notification-configuration --bucket $BUCKET \
  --notification-configuration "{
    \"LambdaFunctionConfigurations\": [{
      \"LambdaFunctionArn\": \"arn:aws:lambda:ap-northeast-2:$ACCT:function:lab-resize\",
      \"Events\": [\"s3:ObjectCreated:*\"],
      \"Filter\": { \"Key\": { \"FilterRules\": [
        { \"Name\": \"prefix\", \"Value\": \"raw/\" }
      ]}}
    }]
  }"
```

## 6. 테스트

```bash
# 큰 image upload
curl -L -o cat.jpg https://placekitten.com/2000/1500
aws s3 cp cat.jpg s3://$BUCKET/raw/cat.jpg

# 5초 대기 후
sleep 5
aws s3 ls s3://$BUCKET/thumb/
# > thumb/cat.jpg

# CloudWatch logs
aws logs tail /aws/lambda/lab-resize --follow --since 5m
```

## 7. X-Ray Service Map

Console → X-Ray → Service Map → lab-resize 의 흐름 확인.

## 8. CloudWatch metric

```bash
aws cloudwatch get-metric-statistics --namespace AWS/Lambda \
  --metric-name Duration --dimensions Name=FunctionName,Value=lab-resize \
  --start-time 2026-05-22T00:00:00Z --end-time 2026-05-22T23:59:59Z \
  --period 60 --statistics Average,Maximum
```

## 9. 부하 — 동시 spike

```bash
for i in $(seq 1 50); do
  aws s3 cp cat.jpg s3://$BUCKET/raw/cat-$i.jpg &
done
wait
```

→ 50 동시 → 50 Lambda concurrent → 모두 처리 (account limit 1000 안).

## 10. Clean up

```bash
aws s3api put-bucket-notification-configuration --bucket $BUCKET \
  --notification-configuration '{}'
aws lambda delete-function --function-name lab-resize
aws iam delete-role-policy --role-name LabResizeRole --policy-name LabResizePolicy
aws iam delete-role --role-name LabResizeRole
aws logs delete-log-group --log-group-name /aws/lambda/lab-resize
aws s3 rb s3://$BUCKET --force
```

## 11. 학습 포인트

- S3 → Lambda async trigger (add-permission + notification-configuration)
- `urllib.parse.unquote_plus` — S3 key URL decoding
- Pillow ARM64 빌드 — manylinux2014_aarch64
- Init phase 에 s3 client (warm 재사용)
- 비용: 1M invocation × 500ms × 512MB = ~$0.40

## 12. 다음

- ▶ [[02-container-image-lambda]]
- ▶ [[03-event-driven-pipeline]]
- ▶ [[../02-triggers/01-async-trigger]]

## 13. 참고

- [[../lambda]]
- [[../05-security/01-execution-role]]
