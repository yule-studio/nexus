---
title: "AWS IoT Analytics"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:23:00+09:00
tags:
  - aws
  - iot
  - analytics
  - iot-analytics
---

# AWS IoT Analytics

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | IoT Analytics hub |

**[[../iot|↑ IoT]]** · **[[../../cloud-aws|↑↑ AWS]]**

> **종료 발표 (2025-12)** — 신규 가입 불가. 기존 분석 = Timestream / Athena / OpenSearch 로 이전.

---

## 1. 한 줄

IoT 데이터 전용 분석 파이프라인. **2025-12 종료** — Kinesis + Timestream / Athena 로 이동 권장.

## 2. 후속

- 실시간 = Kinesis + Lambda / Flink
- 시계열 저장 / 쿼리 = [[../../database/timestream/timestream|Timestream]]
- ad-hoc = [[../../analytics/athena/athena|Athena]] on S3

## 3. 관련

- [[../iot|↑ IoT]]
- [[../../analytics/kinesis/kinesis]]
- [[../../database/timestream/timestream]]
