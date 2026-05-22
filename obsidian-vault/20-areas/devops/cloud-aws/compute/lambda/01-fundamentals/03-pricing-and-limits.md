---
title: "Lambda 01-3 — Pricing + Limits"
kind: knowledge
status: current
created_at: 2026-05-22T16:56:00+09:00
tags: [aws, lambda, fundamentals, pricing, limits]
home_hub: lambda
---

# Lambda Pricing + Limits

> Invocation count + GB-second + Provisioned Concurrency + 외부 (S3 / SQS / etc). hard limit 표.

## 1. Free Tier

| | 무료 |
| --- | --- |
| Invocation | 1,000,000 req/월 |
| Compute | 400,000 GB-s/월 |
| 적용 | 신규 / 기존 계정 모두 |

## 2. 가격 (Seoul region, on-demand)

### Invocation
- $0.20 / 1M requests

### Compute (GB-s)
- ARM64: $0.0000133333 / GB-s
- x86_64: $0.0000166667 / GB-s

> Compute = memory(GB) × duration(s).

### Provisioned Concurrency (PC)
- ARM64: $0.0000033333 / GB-s (idle)
- x86_64: $0.0000041667 / GB-s

(사용 안 해도 24x7 과금)

### SnapStart
- Caching: $0.0000015 / GB-s (snapshot 저장)
- Restore: $0.0001397 / GB (invocation 당)

## 3. 비용 계산 예

**Case 1 — 작은 lambda 1M req/월**:
```
1M req × 100ms × 256MB ARM64
  = 1,000,000 req × 0.1s × 0.25 GB × $0.0000133333
  = 0.33 + (1M - 1M free) × $0.20/1M = $0.33
```

**Case 2 — 큰 lambda 100M req/월 (busy API)**:
```
100M req × 200ms × 512MB ARM64
  = 100,000,000 × 0.2 × 0.5 × $0.0000133333
  = $133
+ (100M - 1M) × $0.20/1M = $19.8
= 약 $153/월
```

같은 traffic ECS Fargate (1 vCPU + 2GB, 2 task 24x7):
```
2 × $36 = $72/월
```

→ 100M req sustained = ECS 가 싸다.

## 4. Hard Limits

| 항목 | Soft limit | Hard limit |
| --- | --- | --- |
| Function timeout | 3s default | **900s (15분)** |
| Memory | 128 MB ~ 10,240 MB | 같음 |
| Ephemeral storage (/tmp) | 512 MB ~ 10 GB | 같음 |
| Deployment zip | 50 MB | **250 MB unzip** |
| Container image | — | **10 GB** |
| Environment variables | 4 KB | 같음 |
| Layers per function | 5 | 같음 |
| Layers total (with code) | — | 250 MB |
| Invocation payload (sync) | 6 MB | 같음 |
| Invocation payload (async) | 256 KB | 같음 |
| Response payload (Function URL stream) | — | 20 MB |
| Concurrent execution / account | 1000 (soft) | 증액 가능 |
| Burst concurrency / minute | 500-3000 (region) | region 따라 |
| Provisioned concurrency / account | 1000 (soft) | 증액 |
| Function name | 64 chars | |
| Description | 256 chars | |

## 5. payload size 우회

응답 6 MB 초과:
- S3 에 저장 → presigned URL 반환
- Function URL 의 response stream (20 MB)
- 작은 chunks 로 split

## 6. timeout 15분 우회

15분 이상 작업:
- Step Functions (Express / Standard) 로 분할
- ECS Fargate (timeout 없음)
- AWS Batch

## 7. Service Quotas — 증액

```bash
aws service-quotas request-service-quota-increase \
  --service-code lambda \
  --quota-code L-B99A9384 \
  --desired-value 5000

# 현재 한도
aws service-quotas get-service-quota \
  --service-code lambda \
  --quota-code L-B99A9384
```

흔한 증액 quota:
- Concurrent executions
- Storage (250 MB)

## 8. 외부 비용

| Service | 비고 |
| --- | --- |
| ECR | image 저장 |
| CloudWatch Logs | invocation log + retention |
| X-Ray | tracing |
| S3 / SQS / SNS / DDB | trigger 의 cost |
| VPC ENI | VPC mode 시 (managed) |
| Data Transfer | egress (NAT 사용 시 큰 부담) |

## 9. Logs 비용 주의

```
Lambda 1M req × 평균 1KB log
  = 1 GB ingestion × $0.50 = $0.5
  + 1 GB stored × $0.03 = $0.03 (월 단위)
```

→ 큰 log volume 시 retention 7-30일 설정 권장.

```bash
aws logs put-retention-policy --log-group-name /aws/lambda/bkurs-fn --retention-in-days 30
```

## 10. 비용 절감 패턴

```
1. ARM64 → -20%
2. Right-sizing memory → 측정 후 최적
3. duration ↓ (algorithm 개선)
4. invocation 묶기 (SQS batch / Kinesis batch)
5. Provisioned Concurrency 만 critical 만
6. Lambda Power Tuning 도구
7. Logs retention 30일
8. X-Ray sampling 5-10%
9. VPC 만 필요할 때만 (cold start ↑)
10. 가끔 호출 → Lambda / 항상 호출 → ECS
```

## 11. Lambda Power Tuning

```bash
git clone https://github.com/alexcasalboni/aws-lambda-power-tuning
# Step Functions deploy → 각 memory 로 N번 invoke → 최적 memory + 비용 자동
```

## 12. SnapStart 비용 vs 효과

```
Java cold start 3000ms → 150ms
SnapStart 비용: cache $0.0000015/GB-s × 60s (publishVersion 후 보존)
             + restore $0.0001397/GB
             
= 거의 무료 (5분 idle 후 시작 시 큰 가치)
```

## 13. 무료 tier 안 쓰는 시나리오

- Provisioned Concurrency 만 활성 → free tier compute 안 쓰임 (PC 는 별도)
- 1 lambda 매분 호출 → 1M req 무료 다 안 씀

## 14. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| sync payload 6MB 초과 | S3 + presigned |
| timeout 15분 초과 | Step Functions |
| concurrency soft limit 1000 초과 → throttle | 증액 / batch / 분산 |
| VPC mode 의 NAT cost | VPC Endpoint |
| Logs 무한 | 30일 retention |
| over-provisioned memory | Power Tuning |
| Provisioned Concurrency 24x7 | spot trigger 또는 ECS |

## 15. 다음

- ▶ [[04-lambda-vs-ecs-vs-stepfn]]
- ▶ [[../03-runtime/01-concurrency-and-scaling]]
- ▶ [[../04-operations/01-observability]]

## 16. 영상 / 자료

| 자료 |
| --- |
| [Lambda Pricing 공식](https://aws.amazon.com/lambda/pricing/) |
| [Lambda Limits](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html) |
| [Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning) |

## 17. 참고

- [[../lambda]]
- [[01-execution-model]]
- [[02-runtime-and-deployment]]
