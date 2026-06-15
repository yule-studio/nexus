---
title: "Amazon Translate"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:14:00+09:00
tags:
  - aws
  - ml
  - translation
---

# Amazon Translate

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Translate hub |

**[[../ml|↑ ML & AI]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

기계 번역 (NMT) — 75+ 언어. real-time + batch + custom terminology / Active Custom Translation.

## 2. 왜 / 언제

- 다국어 콘텐츠 자동 번역 (블로그 / 제품 설명)
- 응용 UI 동적 번역
- 사용자 작성 콘텐츠 (리뷰 / 채팅) 실시간 번역
- 회사 사전 (terminology) 일관성

## 3. 함정 / 대안

- 단가 $15 / 1M characters (real-time) — 누적 시 누적
- 신규 = **Bedrock (Claude / GPT)** 가 맥락 / 톤 우위. 단순 / 대량 = Translate
- Active Custom Translation — 자체 학습 페어로 미세조정

## 4. 관련

- [[../ml|↑ ML & AI]]
- [[../bedrock/bedrock]]
- [[../comprehend/comprehend]]
