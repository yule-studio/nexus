---
title: "AWS IoT Device Management"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:22:00+09:00
tags:
  - aws
  - iot
  - ota
  - device-management
---

# AWS IoT Device Management

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Device Management hub |

**[[../iot|↑ IoT]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

IoT 디바이스 fleet **프로비저닝 / OTA 펌웨어 업데이트 / 모니터링 / 트러블슈팅**. IoT Core 와 짝.

## 2. 기능

| | 의미 |
| --- | --- |
| **Fleet Provisioning** | 자동 등록 (claim cert 또는 JITP) |
| **Jobs** | 그룹 / 개별 디바이스에 명령 (펌웨어 업데이트 등) |
| **Indexing / Search** | 메타 / 상태 기반 검색 |
| **Secure Tunneling** | NAT 뒤 디바이스에 SSH (원격 디버그) |
| **Software Package Catalog** | 펌웨어 버전 관리 |

## 3. 함정

- 디바이스 수만 ~ 수십만 = 진가 발휘. 작은 fleet 은 불필요
- IoT Core 와 함께 활성화

## 4. 관련

- [[../iot|↑ IoT]]
- [[../iot-core/iot-core]]
- [[../greengrass/greengrass]]
