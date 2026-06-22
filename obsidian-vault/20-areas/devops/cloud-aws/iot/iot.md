---
title: "AWS IoT (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:01:00+09:00
tags:
  - aws
  - iot
  - hub
---

# AWS IoT (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | 카테고리 hub |

**[[../cloud-aws|↑ AWS]]**

> IoT 디바이스 클라우드 연결 + 엣지 컴퓨팅 + 산업 / 디지털 트윈.

---

## 1. 서비스

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **IoT Core** | [[iot-core]] | MQTT 브로커 + 디바이스 레지스트리 |
| **IoT Greengrass** | [[greengrass]] | 엣지 컴퓨팅 (디바이스에서 Lambda) |
| **IoT Device Management** | [[device-management]] | 디바이스 프로비저닝 / OTA / 모니터링 |
| **IoT SiteWise** | [[sitewise]] | 산업 장비 데이터 (OPC UA / Modbus) |
| **IoT TwinMaker** | [[twinmaker/twinmaker\|twinmaker]] | 물리 시설 / 장비의 디지털 트윈 (3D scene + 실시간 데이터 binding + Grafana) |
| **IoT Analytics** | [[iot-analytics/iot-analytics\|iot-analytics]] | IoT 데이터 전용 분석 파이프라인 — **2025-12 종료**, Kinesis + Timestream / Athena 로 이전 |
| **IoT Events** | [[events/events\|events]] | IoT 디바이스 상태 머신 + 임계값 알림 — **2025-12 종료**, Step Functions / Lambda 로 이전 |

---

## 2. 표준 흐름

```
디바이스 (센서) → MQTT → IoT Core
   ↓ rule engine
   ├── Lambda (이벤트 처리)
   ├── Kinesis / S3 (적재)
   ├── DynamoDB (상태)
   └── SiteWise (산업 모델링)
```

---

## 3. 관련

- [[../cloud-aws|↑ AWS]]
- [[../analytics/kinesis/kinesis]]
- [[../analytics/firehose/firehose]]
- [[../messaging/sns/sns]] — IoT push 알림
