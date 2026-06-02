---
title: "Amazon Polly"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:13:00+09:00
tags:
  - aws
  - ml
  - tts
  - text-to-speech
---

# Amazon Polly

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Polly hub |

**[[../ml|↑ ML & AI]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

텍스트 → 음성 (TTS). 60+ 언어 / 100+ 음성. Neural / Generative voice (고품질). SSML 지원.

## 2. 왜 / 언제

- IVR / 콜센터 안내 음성
- 응용 안 음성 알림
- 영상 / 광고 더빙 (Generative voice)
- 접근성 (시각 장애 사용자)

## 3. 함정

- 단가 character 당 (Standard $4, Neural $16, Generative $30 / 1M chars)
- 한국어 정확도 양호하지만 ElevenLabs / Azure / OpenAI TTS 와 비교 검토
- Speech Marks 로 lip-sync / 자막 동기화

## 4. 관련

- [[../ml|↑ ML & AI]]
- [[../transcribe/transcribe]] — 반대 방향 (STT)
