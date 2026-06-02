---
title: "AWS IoT TwinMaker"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:25:00+09:00
tags:
  - aws
  - iot
  - digital-twin
  - twinmaker
---

# AWS IoT TwinMaker

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | TwinMaker hub |

**[[../iot|↑ IoT]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

물리 시설 / 장비의 **디지털 트윈** — 3D scene + 실시간 데이터 binding + Grafana plugin 으로 시각화.

## 2. 왜 / 언제

- 공장 / 빌딩 / 발전소의 디지털 트윈
- 운영자가 3D 모델로 장비 상태 확인
- AR / VR 시뮬레이션
- SiteWise + 3D 모델 + Grafana

## 3. 함정

- 3D 모델 / GLTF 준비 필요 (조형 / 변환)
- 사용 빈도 매우 낮음 — 산업 도메인 한정
- NVIDIA Omniverse 등 다른 트윈 솔루션도 검토

## 4. 관련

- [[../iot|↑ IoT]]
- [[../sitewise/sitewise]]
