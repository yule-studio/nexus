---
title: "AWS X-Ray"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:34:00+09:00
tags:
  - aws
  - observability
  - tracing
  - x-ray
---

# AWS X-Ray

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | X-Ray hub |

**[[../dev-tools|↑ Dev Tools]]** · **[[../../cloud-aws|↑↑ AWS]]**

> Observability 의 trace 축이지만 SDK / 디버그 측면에서 Dev Tools 카테고리.

---

## 1. 한 줄

**분산 추적** — 서비스 A → B → C 의 호출 흐름을 trace 로 시각화. 어디서 느리고 어디서 fail 하는지 즉시 파악. OpenTelemetry 와 호환되는 방향으로 진화 중.

## 2. 왜 / 언제

- 마이크로서비스 latency 디버그 (어느 서비스 / 어느 DB 호출이 느린가)
- API Gateway / ALB / Lambda / ECS / RDS / DynamoDB 호출 의존성 맵
- 에러 추적 (어느 단계에서 5xx 발생)
- cold start 추적 (Lambda)
- 응용 코드 안 함수 단위 segment

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Trace** | 한 요청의 전체 흐름 (여러 segment) |
| **Segment** | 한 서비스 안의 작업 |
| **Subsegment** | segment 안의 더 작은 단위 (DB 호출 등) |
| **Service Map** | 서비스 간 호출 의존성 그래프 |
| **Trace Header** | `X-Amzn-Trace-Id` — 호출 사이 전파 |
| **Sampling rule** | 모든 trace X — 1% / rate / 패턴 기반 |
| **Daemon / SDK** | EC2/ECS = X-Ray daemon, Lambda = 자동, OTel collector |

## 4. 표준 응용 통합

| 환경 | 통합 방법 |
| --- | --- |
| Lambda | 함수 설정에서 X-Ray 활성화 (1 클릭) |
| ECS | sidecar 컨테이너 (X-Ray daemon) + SDK |
| EC2 | X-Ray daemon agent + SDK |
| API Gateway | stage 에서 X-Ray 활성화 (1 클릭) |
| ALB | 자동 (Trace-Id header 전파) |
| 응용 코드 | AWS Distro for OpenTelemetry (ADOT) 권장 |

## 5. OTel 호환

AWS 는 **AWS Distro for OpenTelemetry (ADOT)** 를 밀고 있다. 신규 응용은:

```
응용 → OTel SDK → ADOT Collector → X-Ray (또는 Jaeger / Tempo / Datadog)
```

→ vendor-neutral. X-Ray native SDK 는 점차 deprecated 추세.

## 6. 비용 / 함정

- 첫 100,000 trace / 월 무료, 이후 $5 / million trace
- sampling 안 하면 비용 폭발 — 1% sampling 또는 rate-based
- async / fanout (SQS / EventBridge) 의 trace 전파는 명시적 코드 필요
- Lambda cold start 도 segment 로 보임
- **CloudWatch Logs Insights + X-Ray** 조합 — log 와 trace 연결

## 7. 관련

- [[../dev-tools|↑ Dev Tools]]
- [[../../observability/cloudwatch/cloudwatch]]
- [[../../observability/observability]]
- [[../../compute/lambda/lambda]]
