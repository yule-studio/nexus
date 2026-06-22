---
title: "AWS Application Integration (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T10:31:00+09:00
tags:
  - aws
  - integration
  - eventing
  - hub
---

# AWS Application Integration (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | 카테고리 hub |

**[[../cloud-aws|↑ AWS]]**

> 서비스 간 결합도를 낮추는 비동기 / 이벤트 / API gateway 계열. Queue/PubSub 자체는 [[../messaging/messaging|Messaging]] 카테고리.

---

## 1. 서비스

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **EventBridge** | [[eventbridge]] | 이벤트 버스 + 룰 기반 라우팅 + SaaS 통합 + scheduler |
| **API Gateway** | [[../network/api-gateway/api-gateway]] | HTTP / REST / WebSocket API 게이트웨이 |
| **Step Functions** | [[step-functions/step-functions\|step-functions]] | 상태 머신 기반 워크플로우 오케스트레이션 — 250+ 서비스를 코드 없이 재시도 / 분기 / 병렬 / 사람 승인 |
| **AppSync** | [[appsync/appsync\|appsync]] | 매니지드 GraphQL 서버 + WebSocket subscription — DynamoDB / RDS / Lambda / OpenSearch 데이터 소스 통합 |
| **MQ** | [[mq/mq\|mq]] | 매니지드 RabbitMQ / ActiveMQ (AMQP / JMS) — 기존 메시지 브로커 응용을 그대로 AWS 로 |
| **MWAA** | [[mwaa/mwaa\|mwaa]] | 매니지드 Apache Airflow — Python DAG 로 데이터 파이프라인 오케스트레이션 |

---

## 2. 패턴 선택

| 시나리오 | 추천 |
| --- | --- |
| 응용 ↔ 응용 비동기 작업 큐 | SQS ([[../messaging/sqs/sqs]]) |
| 1:N broadcast | SNS ([[../messaging/sns/sns]]) |
| **여러 서비스로 이벤트 라우팅 + 필터링** | EventBridge |
| Cron / schedule (Lambda 호출 등) | EventBridge Scheduler |
| 복잡한 다단계 워크플로우 + 재시도 | Step Functions |
| HTTP / REST API 노출 + 인증 / throttling | API Gateway |
| GraphQL + subscription | AppSync |

---

## 3. SQS / SNS / EventBridge 차이

| | SQS | SNS | EventBridge |
| --- | --- | --- | --- |
| 패턴 | Queue (1:1) | PubSub (1:N) | Event bus + 룰 routing |
| 필터 | X (Group only) | message attribute | **content-based 패턴 매칭** |
| SaaS 통합 | X | X | **Salesforce / Datadog / Zendesk 등 직접** |
| Schedule | X | X | **cron 가능** |
| 비용 | 가장 쌈 | 보통 | 비쌈 (event 당) |

---

## 4. 관련

- [[../cloud-aws|↑ AWS]]
- [[../messaging/messaging]]
- [[../network/network]]
