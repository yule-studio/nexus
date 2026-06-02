---
title: "AWS Elemental MediaLive"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:31:00+09:00
tags:
  - aws
  - media
  - live
  - medialive
---

# AWS Elemental MediaLive

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | MediaLive hub |

**[[../media|↑ Media]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**라이브 인코딩** — RTMP / SRT / HLS 입력 → ABR HLS / DASH 출력. 방송 / 라이브 스트리밍의 인코더.

## 2. 왜 / 언제

- 라이브 방송 (스포츠 / 콘서트 / 뉴스)
- 24/7 채널 (broadcast)
- 다중 비트레이트 ABR 송출
- MediaPackage 와 짝 (인코딩 → 패키징)

## 3. 함정

- 채널 시간당 비용 — 24/7 켜지면 큼 ($30-100+/일)
- standby 비용도 청구 (절감하려면 stop)
- 일반 게임 / 채팅 = **IVS** ([[../ivs/ivs|IVS]]) 가 훨씬 단순 / 저렴

## 4. 관련

- [[../media|↑ Media]]
- [[../mediapackage/mediapackage]]
- [[../ivs/ivs]]
