---
title: "Amazon EventBridge"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T10:49:00+09:00
tags:
  - aws
  - integration
  - eventing
---

# Amazon EventBridge

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | EventBridge hub |

**[[../app-integration|↑ Application Integration]]** · **[[../../cloud-aws|↑↑ AWS]]**

> 옛 이름: **Amazon CloudWatch Events** (2019 rebrand).

---

## 1. 한 줄

**이벤트 버스 + 룰 기반 라우팅 + SaaS 통합 + cron scheduler**. 응용 / AWS / SaaS 의 이벤트를 받아 Lambda / SQS / SNS / Step Functions 등 20+ 타깃으로 흘린다.

---

## 2. 왜 / 언제

- 여러 서비스로 이벤트 broadcast + **내용 기반 필터**
- AWS 자체 이벤트에 자동 반응 (예: EC2 상태 변경 / S3 객체 생성 / Auto Scaling 이벤트)
- **cron / schedule** — Lambda 를 정기 호출 (예: 매 5분, 매 자정)
- SaaS (Datadog / Zendesk / Salesforce / Auth0 등) 의 이벤트 수신
- 서비스 간 결합도 ↓ (응용 A 가 응용 B 의 존재를 모름)

---

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Event Bus** | 이벤트 통로. `default` / **custom** / **partner** (SaaS) |
| **Event** | JSON `{source, detail-type, detail, ...}` |
| **Rule** | event pattern 매칭 또는 schedule. 매칭 시 target 호출 |
| **Target** | Lambda / SQS / SNS / Step Functions / API Destination / ECS Task / ... (20+) |
| **Pipes** | source → filter/enrich → target (Lambda 없이 streaming integration) |
| **Scheduler** | EventBridge Scheduler — 진화된 cron (cron / rate / one-time) |
| **Archive / Replay** | 이벤트 보관 + 재실행 |
| **Schema Registry** | 이벤트 스키마 자동 발견 + 코드 생성 |

---

## 4. 표준 패턴

### 4.1 응용 이벤트 라우팅

```
응용 → put_events → custom bus
            ↓ Rule (source=myapp, detail-type=UserSignedUp)
       ┌──────────────┬───────────────┐
       Lambda         SQS             Step Functions
```

```python
events.put_events(Entries=[{
    "Source": "myapp",
    "DetailType": "UserSignedUp",
    "Detail": json.dumps({"userId": "u_123", "email": "x@y"}),
    "EventBusName": "myapp-bus",
}])
```

### 4.2 cron — Lambda 정기 실행

```yaml
ScheduleRule:
  Type: AWS::Events::Rule
  Properties:
    ScheduleExpression: "cron(0 2 * * ? *)"   # 매일 02:00 UTC
    Targets:
      - Arn: !GetAtt CleanupFn.Arn
        Id: cleanup
```

→ Lambda 의 schedule 트리거 자체가 EventBridge.

### 4.3 AWS 자체 이벤트 자동 반응

```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": { "state": ["terminated"] }
}
```

→ EC2 종료 시 Lambda 가 알림 / 정리.

---

## 5. EventBridge vs SNS vs SQS

| | SQS | SNS | **EventBridge** |
| --- | --- | --- | --- |
| 패턴 | Queue | PubSub | Event bus + 룰 |
| 필터 | X | message attribute | **content (JSON path 매칭)** |
| target 수 | 1 (queue) | 다수 (subscription) | 5 / rule |
| SaaS 통합 | X | X | **직접 통합** |
| Schedule | X | X | **cron / rate** |
| 비용 | 가장 쌈 | 중간 | event 당 $1/M (비쌈) |
| latency | μs | ms | 0.5초~ |

→ 단순 fan-out = SNS. **content 필터 / SaaS / schedule** = EventBridge.

---

## 6. 함정 / 비용

- **$1 / million events** (custom bus). 대량이면 비쌈.
- **at-least-once** — Lambda target 은 멱등 (idempotent) 필요.
- **target 5 개 한계 / rule** — 더 필요하면 SNS 거치기.
- **payload 256 KB** — 더 크면 S3 + reference.
- **순서 보장 X** — 순서 필요하면 SQS FIFO / Kinesis.
- **Schedule = EventBridge Scheduler** (별도 서비스, 더 진화). 신규는 Scheduler 권장.

---

## 7. 관련

- [[../app-integration|↑ Application Integration]]
- [[../../messaging/sns/sns]]
- [[../../messaging/sqs/sqs]]
- [[../../compute/lambda/lambda]]
