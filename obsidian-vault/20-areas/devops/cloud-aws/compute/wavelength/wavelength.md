---
title: "AWS Wavelength"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:02:00+09:00
tags:
  - aws
  - compute
  - 5g
  - wavelength
---

# AWS Wavelength

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Wavelength hub |

**[[../compute|↑ Compute]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**통신사 5G 망 안에 AWS 컴퓨팅 배치** — 모바일 디바이스 ↔ AWS 의 latency 가 5G base station 한 홉. Verizon / KDDI / SK Telecom 등.

## 2. 왜 / 언제

- 5G 모바일 응용의 ultra-low latency (< 10ms)
- AR / VR / 게임 / 자율주행
- 모바일 엣지 컴퓨팅 (MEC)
- 사용 빈도 매우 낮음 — 5G 응용 한정

## 3. 함정

- 한국 = SK Telecom 제휴 (region 확인)
- 일반 응용은 Local Zones / region 으로 충분
- 진짜 5G MEC 요구 사항 있을 때만

## 4. 관련

- [[../compute|↑ Compute]]
- [[../local-zones/local-zones]]
- [[../outposts/outposts]]
