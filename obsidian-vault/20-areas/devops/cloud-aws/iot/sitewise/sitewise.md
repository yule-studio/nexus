---
title: "AWS IoT SiteWise"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:24:00+09:00
tags:
  - aws
  - iot
  - industrial
  - sitewise
---

# AWS IoT SiteWise

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | SiteWise hub |

**[[../iot|↑ IoT]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

산업 장비 (OPC UA / Modbus / EthernetIP) 데이터 수집 + **자산 모델링** (asset hierarchy + computed metric). 공장 / 발전소 / 빌딩.

## 2. 왜 / 언제

- 공장 PLC / SCADA 데이터를 AWS 로
- 디지털 트윈 의 데이터 소스 ([[../twinmaker/twinmaker|TwinMaker]])
- 시계열 + 자산 계층 ("라인 1 > 모터 A > 온도")
- KPI computed metric (OEE / availability)

## 3. 함정

- 산업 도메인 학습 필요 (OPC UA 등)
- 일반 IoT = IoT Core 가 더 단순
- SiteWise Edge — 공장 게이트웨이

## 4. 관련

- [[../iot|↑ IoT]]
- [[../iot-core/iot-core]]
- [[../twinmaker/twinmaker]]
