---
title: "Amazon MSK (Managed Streaming for Apache Kafka)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:46:00+09:00
tags:
  - aws
  - analytics
  - streaming
  - kafka
  - msk
---

# Amazon MSK

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | MSK hub |

**[[../analytics|↑ Analytics]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **Apache Kafka**. broker / Zookeeper (또는 KRaft) AWS 가 운영, 사용자는 topic / producer / consumer 만. Kinesis 보다 표준 / 큰 throughput / 멀티클라우드 이식성.

## 2. 왜 / 언제

- 기존 Kafka 응용을 그대로 AWS 로
- 매우 높은 throughput (수십만 / 초)
- 엄격한 ordering (partition 내)
- Kafka 생태계 (Kafka Connect / Streams / KSQL) 사용
- 멀티클라우드 이식성 (Kinesis 는 AWS lock-in)

## 3. 두 가지 배포

| | Provisioned | Serverless |
| --- | --- | --- |
| 모델 | broker 노드 직접 | 용량 자동 |
| Setup | broker 수 / 타입 / 스토리지 | partition / topic 만 |
| 비용 | broker 시간 + EBS | 시간당 cluster + partition 수 + 데이터 |
| Replication | 직접 설정 | 자동 |
| Tiered Storage | 옵션 (저렴한 cold tier) | 자동 |

## 4. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Topic** | 메시지 채널 (partition 단위) |
| **Partition** | 병렬 + ordering 단위 |
| **Producer / Consumer** | 송/수신 응용 |
| **Consumer Group** | 분산 consumer (offset 공유) |
| **Schema Registry (Glue)** | Avro / JSON 스키마 |
| **MSK Connect** | 매니지드 Kafka Connect |
| **IAM Auth** | broker 접근 IAM 권한으로 |
| **TLS / SASL** | 인증 옵션 |

## 5. MSK vs Kinesis Streams

| | MSK | Kinesis Streams |
| --- | --- | --- |
| 프로토콜 | Kafka 표준 | Kinesis API |
| Setup | 클러스터 운영 (Serverless 도 가능) | shard 만 |
| Ecosystem | Kafka Connect / Streams / KSQL | Lambda / Firehose / Flink |
| Lock-in | 낮음 (Kafka 표준) | 높음 (AWS) |
| 비용 | broker 시간 (높음) | shard 시간 (적은~중) |
| 처리량 | 매우 높음 | 높음 |

→ 표준 Kafka / 이식성 = MSK. AWS native / 단순 = Kinesis.

## 6. 비용 / 함정

- Provisioned: broker 시간 ~$0.21/시간 + EBS — **3 broker 최소면 월 $450~**
- Serverless: $0.75 / 시간 / cluster (24h = $540/월) + 사용량
- Cross-AZ 데이터 transfer 비싸므로 producer/consumer 를 같은 AZ
- 큰 partition 수 (수천) 는 broker 메모리 압박
- IAM Auth 권장 (SCRAM 보다)

## 7. 관련

- [[../analytics|↑ Analytics]]
- [[../kinesis/kinesis]]
- [[../firehose/firehose]]
- [[../../../computer-science/network/rpc-messaging/mqtt-amqp-kafka|↗ Kafka 이론]]
