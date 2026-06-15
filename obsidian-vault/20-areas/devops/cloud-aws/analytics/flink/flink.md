---
title: "Amazon Managed Service for Apache Flink"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:04:00+09:00
tags:
  - aws
  - analytics
  - streaming
  - flink
---

# Amazon Managed Service for Apache Flink

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Flink hub |

**[[../analytics|↑ Analytics]]** · **[[../../cloud-aws|↑↑ AWS]]**

> 옛 이름: **Kinesis Data Analytics for Apache Flink** (2023 rebrand).

---

## 1. 한 줄

매니지드 **Apache Flink** — 실시간 스트림 처리 (Java / Python / SQL). Kinesis / MSK / Kafka 의 consumer 로 stateful 처리 / 윈도잉 / join.

## 2. 왜 / 언제

- Kinesis / MSK 스트림에서 실시간 집계 / 패턴 매칭
- exactly-once 보장 필요한 스트림 처리
- Lambda 보다 stateful / 큰 처리량
- SQL 로 간단한 스트림 처리 (Flink SQL)

## 3. 함정

- Flink 학습 곡선 (window / state / checkpoint)
- KPU 시간당 비용 — idle 도 청구
- 단순 처리 = Lambda, Kinesis consumer. **stateful 복잡** = Flink

## 4. 관련

- [[../analytics|↑ Analytics]]
- [[../kinesis/kinesis]] / [[../msk/msk]]
- [[../firehose/firehose]]
