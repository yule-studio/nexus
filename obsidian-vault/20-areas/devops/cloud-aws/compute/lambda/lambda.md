---
title: "AWS Lambda — Serverless Function"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:25:00+09:00
tags:
  - aws
  - compute
  - lambda
  - serverless
---

# AWS Lambda

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Lambda 개념 + 사용 |

**[[compute|↑ Compute]]** · **[[cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

이벤트가 오면 **함수를 띄워 실행**, 끝나면 종료. 서버 관리 0. 실행 시간만 과금.

---

## 2. 왜

- **서버 띄울 일 X** — 사용 시간만
- **자동 확장** — 1 → 수천 invocation 동시
- **event-driven 자연스러움** — S3 upload → Lambda 등 통합
- 작은 작업 / 가끔 호출 = EC2/ECS 보다 저렴

한계:
- **15 분 실행 한계**
- **cold start** (수백 ms 첫 호출)
- **stateless** (in-memory 보존 X)
- **256 MB ~ 10 GB RAM** (가격 비례)

대안:
- **ECS / EKS** — 긴 / 큰 작업
- **Step Functions** — 여러 Lambda chain
- **App Runner / EC2** — 일반 응용

---

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Function** | 코드 + 설정 + role |
| **Runtime** | Python / Node / Java / Go / Ruby / .NET / custom |
| **Handler** | entry point (`handler.lambda_handler`) |
| **Trigger** | S3 / API Gateway / EventBridge / SQS / DynamoDB Stream / ... |
| **Layer** | 공유 라이브러리 |
| **Provisioned Concurrency** | cold start 회피 (warm pool) |
| **Reserved Concurrency** | 최대 동시 실행 한정 |
| **VPC Lambda** | VPC 안 자원 (RDS 등) 접근 |

---

## 4. 설치 / 첫 함수

### 4.1 Console

```python
# handler.py
def lambda_handler(event, context):
    return {"statusCode": 200, "body": "hello"}
```

Console → Create function → Author from scratch → Python 3.12.

### 4.2 CLI / zip

```bash
# 작성
mkdir myfn && cd myfn
echo 'def lambda_handler(event, context): return "hi"' > handler.py

zip -r ../myfn.zip .

aws lambda create-function \
  --function-name myfn \
  --runtime python3.12 \
  --role arn:aws:iam::111222333444:role/lambda-basic-role \
  --handler handler.lambda_handler \
  --zip-file fileb://../myfn.zip
```

업데이트:
```bash
aws lambda update-function-code --function-name myfn --zip-file fileb://myfn.zip
```

### 4.3 SAM / CDK / Serverless Framework

```yaml
# template.yaml (SAM)
Resources:
  MyFn:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ./
      Handler: handler.lambda_handler
      Runtime: python3.12
      Events:
        Api:
          Type: Api
          Properties:
            Path: /hello
            Method: get
```

```bash
sam build && sam deploy --guided
```

### 4.4 Container Image (10 GB 까지)

```dockerfile
FROM public.ecr.aws/lambda/python:3.12
COPY handler.py ${LAMBDA_TASK_ROOT}
CMD ["handler.lambda_handler"]
```

```bash
docker build -t myfn .
aws ecr ... # tag + push
aws lambda create-function --package-type Image --code ImageUri=...
```

---

## 5. Trigger 예

### 5.1 API Gateway (HTTP)

```yaml
Events:
  Api:
    Type: HttpApi
    Properties:
      Path: /
      Method: POST
```

→ POST https://abc.execute-api... → Lambda 실행.

### 5.2 S3 (object created)

```python
def lambda_handler(event, context):
    for r in event["Records"]:
        bucket = r["s3"]["bucket"]["name"]
        key    = r["s3"]["object"]["key"]
        # 이미지 thumbnail 생성 등
```

### 5.3 SQS

```python
def lambda_handler(event, context):
    for r in event["Records"]:
        body = r["body"]
        # 처리
```

Lambda 가 자동 polling. 동시 배치 size 설정.

### 5.4 EventBridge cron

```yaml
Events:
  Schedule:
    Type: Schedule
    Properties:
      Schedule: rate(5 minutes)
      # 또는 cron(0 8 * * ? *)
```

### 5.5 DynamoDB Stream

테이블 변경 → Lambda (CDC).

### 5.6 CloudWatch Logs / Kinesis / SNS / Cognito / ...
모두 지원.

---

## 6. Cold Start

```
첫 호출 (또는 idle 후):
  download → init → handler 실행
  Python / Node: 200-500 ms
  Java / .NET:   1-3 초
  GraalVM native: 100 ms
```

해결:
- **Provisioned Concurrency** — warm pool 유지 (추가 비용)
- **SnapStart** (Java 11/17/21) — 90% 절감
- 작은 deployment package
- VPC 안 = ENI 생성 ~ 옛 10s+ (현재 < 1s 으로 개선)

---

## 7. Memory / CPU

```
Memory 128 MB ~ 10 GB
CPU = memory 비례 (1769 MB 부터 1 vCPU)
```

→ memory 늘리면 CPU도 ↑ → 더 빠름 + 비싸지 않을 수도 (실행 시간 ↓).
**Power Tuning** 도구로 최적 memory 찾기.

---

## 8. Concurrency

- **Account limit**: 1000 (기본, 증가 요청 가능)
- **Reserved**: 함수별 최대 (다른 함수 보호)
- **Provisioned**: warm
- 한 함수 burst = ~ 500-3000 (region 별)

---

## 9. 비용

```
$0.0000166667 / GB·s + $0.20 / 1M 요청
```

예 — 512 MB 함수, 200 ms, 월 1M 호출:
```
GB·s = 0.5 × 0.2 × 1M = 100,000
$ = 100K × $0.0000166667 + 1M × $0.20/1M = $1.67 + $0.20 = ~$1.87
```

매우 저렴. but 항상 켜둘 = ECS 더 저렴할 수도.

---

## 10. 사용 시나리오

- API backend (작은 / 가끔)
- 파일 처리 (S3 upload → thumbnail)
- ETL / data pipeline
- cron job
- ChatBot / Webhook
- IoT
- Cognito trigger (사용자 가입 후 처리)

부적합:
- 15 분 초과
- WebSocket persistent connection (API Gateway WebSocket 별도)
- 매우 큰 RAM / GPU
- 매우 sustained 트래픽 (ECS 가 더 쌈)

---

## 11. 함정

### 11.1 timeout / memory 부족
default 3 초 / 128 MB. 늘리기.

### 11.2 cold start 의 latency
P99 latency 폭증. Provisioned / SnapStart / smaller package.

### 11.3 VPC Lambda 의 ENI
옛 — 큰 cold start. 새 — Hyperplane 으로 해결. 단, IP 소모 ↑.

### 11.4 packaging 의 크기
unzipped 250 MB / zipped 50 MB 한계. Layer 또는 container image (10 GB).

### 11.5 logging — CloudWatch Logs 만
log 폭증 → 비용. log retention 설정.

### 11.6 retry — async 호출
S3 / SNS 등 async = 자동 재시도. DLQ (Dead-Letter Queue) 설정 필수.

### 11.7 idempotency
재시도 시 중복 처리 위험. event ID + dedup.

---

## 12. 학습 자료

- AWS Lambda docs
- **Serverless Patterns** — serverlessland.com
- **Lambda Power Tuning** — github.com/alexcasalboni

---

## 13. 관련

- [[compute]] — Compute hub
- [[ec2]] / [[ecs]] — 대안
- [[sqs]] — 자주 함께
- [[cloudwatch]] — 로그
