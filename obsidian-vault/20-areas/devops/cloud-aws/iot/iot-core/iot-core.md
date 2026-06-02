---
title: "AWS IoT Core"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:20:00+09:00
tags:
  - aws
  - iot
  - mqtt
  - iot-core
---

# AWS IoT Core

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | IoT Core hub |

**[[../iot|↑ IoT]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **MQTT 브로커 + 디바이스 레지스트리 + 룰 엔진**. IoT 디바이스 클라우드 연결의 중심.

## 2. 핵심 / 함정

| 개념 | 의미 |
| --- | --- |
| **Thing** | 디바이스 (인증서 + 메타) |
| **Topic** | MQTT pub/sub 채널 |
| **Rule Engine** | SQL like 룰 → Lambda / DynamoDB / S3 / Kinesis 로 라우팅 |
| **Device Shadow** | 디바이스 의 마지막 상태 (오프라인 때도) |
| **X.509 인증** | per-device 인증서 |
| **함정** | MQTT 5.0 / QoS / TLS 학습 곡선. 디바이스 인증서 회전 자동화 필요 |

## 3. 관련

- [[../iot|↑ IoT]]
- [[../greengrass/greengrass]]
- [[../device-management/device-management]]
- [[../../analytics/kinesis/kinesis]]
