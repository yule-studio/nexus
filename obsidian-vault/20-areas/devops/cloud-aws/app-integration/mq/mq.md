---
title: "Amazon MQ"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:35:00+09:00
tags:
  - aws
  - integration
  - rabbitmq
  - activemq
---

# Amazon MQ

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | MQ hub |

**[[../app-integration|↑ Application Integration]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **RabbitMQ / ActiveMQ** (AMQP / MQTT / STOMP / OpenWire / JMS). 기존 RabbitMQ / ActiveMQ 응용을 그대로 AWS 로.

## 2. 왜 / 언제

- 기존 AMQP / JMS 응용을 옮길 때 (SQS 로 코드 리팩토링 부담 회피)
- AMQP 가 필요 (SQS 가 안 됨)
- 신규 응용은 SQS / SNS / EventBridge 권장

## 3. 함정

- broker EC2 시간당 청구 — 사용량 적어도 항상 비용
- 신규 = SQS / SNS / EventBridge / MSK (Kafka)
- AMQP 1.0 / JMS 가 진짜 필요한지 검토 후 결정

## 4. 관련

- [[../app-integration|↑ Application Integration]]
- [[../../messaging/sqs/sqs]]
- [[../../analytics/msk/msk]]
