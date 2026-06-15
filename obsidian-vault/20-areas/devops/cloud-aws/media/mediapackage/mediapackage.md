---
title: "AWS Elemental MediaPackage"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:32:00+09:00
tags:
  - aws
  - media
  - packaging
  - mediapackage
---

# AWS Elemental MediaPackage

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | MediaPackage hub |

**[[../media|↑ Media]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

라이브 / VOD **패키징 + DRM + 광고 마커**. 인코딩된 stream → HLS / DASH / CMAF / MSS 출력 + CloudFront 까지.

## 2. 왜 / 언제

- 다중 디바이스 (TV / 모바일 / 웹) 호환 packaging
- DRM (Widevine / FairPlay / PlayReady)
- 시간 이동 (catch-up, time-shift)
- live origin + harvest (라이브를 VOD 로)

## 3. 함정

- GB egress + ingest 비용 — 큰 stream 비싸짐
- MediaLive + MediaPackage + CloudFront 표준 조합
- 단순 라이브 = IVS

## 4. 관련

- [[../media|↑ Media]]
- [[../medialive/medialive]]
- [[../../network/cloudfront/cloudfront]]
