---
title: "Amazon IVS (Interactive Video Service)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:35:00+09:00
tags:
  - aws
  - media
  - live
  - ivs
---

# Amazon IVS

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | IVS hub |

**[[../media|↑ Media]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **저지연 라이브 스트리밍** (< 3s) + 채팅. Twitch 의 엔진 기반. RTMP 입력 → 글로벌 viewers. MediaLive + MediaPackage 보다 단순.

## 2. 왜 / 언제

- 게임 방송 / 라이브 쇼핑 / 인터랙티브 라이브
- 채팅 + 스트리밍 통합 (Real-Time Streaming + chat)
- 빠른 시작 (수 분, 채널 SDK)
- Real-Time Streaming (멀티 host 화상 라이브)

## 3. 함정

- 시간 / GB 비용 — viewer 많아지면 누적
- DRM / 광고 = MediaLive + MediaPackage + MediaTailor 가 필요할 수도
- 한국 region 지원 — 일본 region 통해 서비스 가능

## 4. 관련

- [[../media|↑ Media]]
- [[../medialive/medialive]]
- [[../mediapackage/mediapackage]]
