---
title: "AWS IoT Events"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:26:00+09:00
tags:
  - aws
  - iot
  - state-machine
  - iot-events
---

# AWS IoT Events

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | IoT Events hub |

**[[../iot|↑ IoT]]** · **[[../../cloud-aws|↑↑ AWS]]**

> **종료 발표 (2025-12)** — 신규 가입 불가.

---

## 1. 한 줄

IoT 디바이스 상태 머신 (state machine) — 임계값 기반 알림 / 액션. **2025-12 종료** — Step Functions / Lambda / Timestream + 자체 룰로 이전.

## 2. 후속

- 상태 머신 = [[../../app-integration/step-functions/step-functions|Step Functions]]
- 단순 룰 = IoT Core Rule Engine + Lambda
- 시계열 임계값 = CloudWatch / Timestream + 알람

## 3. 관련

- [[../iot|↑ IoT]]
- [[../iot-core/iot-core]]
- [[../../app-integration/step-functions/step-functions]]
