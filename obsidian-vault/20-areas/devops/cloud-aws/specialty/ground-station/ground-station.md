---
title: "AWS Ground Station"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:44:00+09:00
tags:
  - aws
  - specialty
  - satellite
  - ground-station
---

# AWS Ground Station

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Ground Station hub |

**[[../specialty|↑ Specialty]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

위성 안테나 **as a Service** — 분 단위로 AWS 의 글로벌 안테나 빌려 위성 데이터 수신 / 명령 송신. NASA / Capella Space / Planet 등.

## 2. 함정

- 사용 빈도 매우 낮음 — 실제 위성 운영 조직만
- contact (수신 / 송신) 시간당 비용
- 한국 region 미지원 — 글로벌 안테나 활용

## 3. 관련

- [[../specialty|↑ Specialty]]
- [[../../storage/s3/s3]] — 수신 데이터
