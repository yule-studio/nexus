---
title: "AWS Messaging (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:10:00+09:00
tags:
  - aws
  - messaging
  - hub
---

# AWS Messaging (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 카테고리 hub |

**[[../cloud-aws|↑ AWS]]**

---

## 1. 서비스

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **SQS** | [[sqs]] | Queue (1:1, at-least-once) |
| **SNS** | [[sns]] | Pub/Sub (1:N fan-out) |
| **SES** | [[ses/ses\|ses]] | 매니지드 이메일 발송 / 수신 — 가입 인증 / 비밀번호 재설정 / 알림 메일 백엔드 |

---

## 2. 모델 비교

| | SQS | SNS | EventBridge | Kinesis | MSK |
| --- | --- | --- | --- | --- | --- |
| 패턴 | Queue | Pub/Sub | Event bus + routing | Stream | Kafka |
| 순서 | FIFO 옵션 | X | X | shard 내 | partition 내 |
| 영속 | 14일까지 | X | X | 7-365 일 | 영구 |
| 처리량 | 거대 | 거대 | 보통 | 매우 큼 | 매우 큼 |

→ 작은 작업 큐 = SQS. broadcast = SNS. Kafka 호환 / 대량 streaming = MSK / Kinesis.

---

## 3. 표준 패턴 — Fan-out

```
Producer → SNS topic → SQS queue 1 → Worker 1
                    → SQS queue 2 → Worker 2
                    → Lambda
```

SNS = broadcast, SQS = 각 consumer 의 큐 (worker 가 자기 속도로).

---

## 4. 관련

- [[../cloud-aws|↑ AWS]]
- [[../../../computer-science/network/rpc-messaging/mqtt-amqp-kafka|↗ MQ/Kafka 이론]]
