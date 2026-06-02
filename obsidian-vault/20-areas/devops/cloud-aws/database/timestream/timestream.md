---
title: "Amazon Timestream"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:23:00+09:00
tags:
  - aws
  - database
  - timeseries
---

# Amazon Timestream

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Timestream hub |

**[[../database|↑ Database]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **시계열 DB**. IoT 텔레메트리 / 응용 메트릭 / 모니터링 데이터. memory tier + magnetic tier 자동 lifecycle.

## 2. 왜 / 언제

- IoT / 센서 데이터 (수백만 데이터 포인트 / 초)
- 자체 InfluxDB / TimescaleDB 운영 부담 회피
- SQL 호환 (시계열 함수)
- Grafana / QuickSight 직접 연결

## 3. 함정

- 데이터 모델이 시계열 특화 — 일반 RDB 처럼 쓰면 어색
- 최근 데이터 vs 오래된 데이터 tier 자동 — 비용 효율
- 대안: **Timestream for InfluxDB** (2024+, InfluxDB OSS 호환) — InfluxQL / Flux 사용자에게 더 친숙
- CloudWatch Metrics 가 적절한 규모일 수도

## 4. 관련

- [[../database|↑ Database]]
- [[../../iot/iot]]
- [[../../observability/cloudwatch/cloudwatch]]
