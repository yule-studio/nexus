---
title: "Amazon Transcribe"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:12:00+09:00
tags:
  - aws
  - ml
  - speech-to-text
  - stt
---

# Amazon Transcribe

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Transcribe hub |

**[[../ml|↑ ML & AI]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

음성 → 텍스트 (STT). batch / streaming / Medical / Call Analytics. 100+ 언어 (한국어 포함).

## 2. 왜 / 언제

- 콜센터 녹취 → 텍스트 + 감정 분석
- 영상 자막 자동 생성
- 회의 노트 자동화
- Medical — 진료 음성 → 차트

## 3. 함정 / 대안

- 단가 분 단위 ($0.024 ~ 0.063 / 분)
- 한국어 정확도 — OpenAI Whisper / Bedrock Claude STT 비교 검토
- speaker diarization / vocabulary customization 무료 옵션

## 4. 관련

- [[../ml|↑ ML & AI]]
- [[../polly/polly]] — 반대 방향 (TTS)
- [[../comprehend/comprehend]] — 텍스트 분석 단계
