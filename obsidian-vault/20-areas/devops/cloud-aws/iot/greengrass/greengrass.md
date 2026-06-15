---
title: "AWS IoT Greengrass"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:21:00+09:00
tags:
  - aws
  - iot
  - edge
  - greengrass
---

# AWS IoT Greengrass

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Greengrass hub |

**[[../iot|↑ IoT]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**엣지 컴퓨팅** — 디바이스 / 게이트웨이 위에서 Lambda / ML 추론 / 로컬 MQTT 브로커 실행. 오프라인 동작 + 클라우드 동기화.

## 2. 왜 / 언제

- 네트워크 불안정 — 오프라인 동작 필요
- 클라우드 round-trip latency 가 큰 응용
- ML 추론을 엣지에서 (대역폭 절감)
- 디바이스 fleet (수천 ~ 수만) 통합 관리

## 3. 함정

- 엣지 디바이스 자체 운영 부담 (HW / OS / 보안 패치)
- Greengrass v1 → v2 마이그레이션 (v2 권장)
- Snowball Edge 와 보완 / 경쟁

## 4. 관련

- [[../iot|↑ IoT]]
- [[../iot-core/iot-core]]
- [[../../compute/lambda/lambda]]
