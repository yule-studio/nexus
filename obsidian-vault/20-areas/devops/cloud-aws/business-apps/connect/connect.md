---
title: "Amazon Connect"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:14:00+09:00
tags:
  - aws
  - business-apps
  - call-center
  - connect
---

# Amazon Connect

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Connect hub |

**[[../business-apps|↑ Business Apps & EUC]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **클라우드 콜센터** — 인입 / 분기 / 큐 / 상담사 / 녹취 / 분석. Genesys / Twilio Flex 대안. Lex 챗봇 / Lambda 통합.

## 2. 왜 / 언제

- 사내 콜센터 SaaS 로 (PBX 직접 운영 X)
- IVR (Lex / Lambda 로 자유롭게)
- 옴니채널 (음성 / 채팅 / 작업)
- Contact Lens — 통화 transcript + 감정 분석

## 3. 함정

- 시간당 + 분당 + 메시지당 비용 — 작은 콜센터엔 비쌀 수도
- Contact Flow (IVR 디자이너) 학습 곡선
- 대안: Twilio / Genesys / 한국 = 카카오엔터프라이즈 등

## 4. 관련

- [[../business-apps|↑ Business Apps & EUC]]
- [[../../ml/lex/lex]] — IVR 챗봇
- [[../../ml/transcribe/transcribe]] — 통화 녹취
- [[../chime/chime]]
