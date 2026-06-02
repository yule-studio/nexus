---
title: "AWS Elemental MediaTailor"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:34:00+09:00
tags:
  - aws
  - media
  - ads
  - mediatailor
---

# AWS Elemental MediaTailor

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | MediaTailor hub |

**[[../media|↑ Media]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**SSAI (Server-Side Ad Insertion)** — 비디오 stream 에 서버 측으로 광고 자동 삽입. AdBlocker 회피 + 개인화 광고.

## 2. 왜 / 언제

- 광고 기반 OTT / FAST 채널
- live + VOD 광고 삽입
- VAST / VMAP 광고 서버 통합
- channel assembly — 가상 채널 (24/7 EPG)

## 3. 함정

- 분석 / 광고 결정 = 자체 ad server 또는 SaaS (Google Ad Manager 등)
- 작은 운영 = 클라이언트 측 광고 삽입 (CSAI) 도 검토
- 미디어 도메인 학습 필요

## 4. 관련

- [[../media|↑ Media]]
- [[../mediapackage/mediapackage]]
- [[../medialive/medialive]]
