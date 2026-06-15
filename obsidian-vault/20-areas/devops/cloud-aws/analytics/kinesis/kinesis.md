---
title: "Amazon Kinesis Data Streams"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:43:00+09:00
tags:
  - aws
  - analytics
  - streaming
  - kinesis
---

# Amazon Kinesis Data Streams

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Kinesis hub |

**[[../analytics|↑ Analytics]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

실시간 **스트림** (Kafka 같음). shard 기반 + 7~365일 retention + Lambda / Firehose / Flink consumer.

## 2. 왜 / 언제

- 응용 / 디바이스 / 로그의 실시간 수집
- 여러 consumer 가 같은 stream 재생 (replay 가능)
- Lambda 트리거 (배치 batch size + parallel)
- Click-stream / IoT 텔레메트리 / 게임 이벤트

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Stream** | 데이터 채널 |
| **Shard** | 단위 — 1MB/s write, 2MB/s read, 1000 records/s write |
| **Partition Key** | record → shard 매핑 |
| **Sequence Number** | record 순서 |
| **Retention** | 1-365 일 (기본 24h) |
| **On-demand mode** | shard 자동 확장 (편함, 비쌈) |
| **Provisioned mode** | shard 수 직접 (싸지만 관리) |
| **EFO (Enhanced Fan-Out)** | consumer 마다 2MB/s 보장 |

## 4. Kinesis vs SQS vs MSK

| | Kinesis | SQS | MSK (Kafka) |
| --- | --- | --- | --- |
| 패턴 | 스트림 | 큐 | 스트림 |
| 순서 | shard 내 보장 | FIFO 옵션 | partition 내 |
| Replay | ✅ retention 안 | X | ✅ |
| consumer 수 | 다수 (EFO 권장) | 1 (or fan-out via SNS) | 다수 |
| 비용 | shard 시간당 | message 당 | broker 시간당 |

→ 단일 작업 큐 = SQS. **재생 / 다수 consumer / 순서** = Kinesis. Kafka 호환 = MSK.

## 5. 표준 통합

```
Producer (응용 / agent)
   ↓ PutRecords
Kinesis Stream
   ├── Lambda (이벤트 처리)
   ├── Firehose → S3 (저장)
   └── Flink / Spark Streaming (집계)
```

## 6. 비용 / 함정

- Provisioned: shard 시간당 $0.015 + record 당 추가
- On-demand: $0.04 / GB ingest + $0.04 / GB egress
- **hot partition key** (같은 key 에 트래픽 몰림) → shard throttle
- **Lambda consumer** — batch / parallelization factor 조정
- **EFO** 비쌈 — 일반 consumer 는 polling 으로 충분
- retention 길게 잡으면 비용 ↑

## 7. 관련

- [[../analytics|↑ Analytics]]
- [[../firehose/firehose]]
- [[../msk/msk]]
- [[../../compute/lambda/lambda]]
