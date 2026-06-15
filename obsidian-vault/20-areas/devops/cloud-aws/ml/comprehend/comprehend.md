---
title: "Amazon Comprehend"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:10:00+09:00
tags:
  - aws
  - ml
  - nlp
---

# Amazon Comprehend

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Comprehend hub |

**[[../ml|↑ ML & AI]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

NLP API — **감정 / 개체 / 키 구문 / 언어 / 분류 / 토픽 모델링 + PII 검출**. Comprehend Medical 은 의료 특화.

## 2. 왜 / 언제

- 짧은 텍스트 감정 / 분류 (리뷰 / 티켓)
- 개체 추출 (사람 / 조직 / 위치)
- PII 자동 마스킹
- Medical — 진료 노트 ICD 코드

## 3. 함정 / 대안

- **신규 응용은 Bedrock (Claude)** 가 정확도 / 유연성 우위
- 단가 (10K characters 당) 누적되면 LLM 보다 비쌀 수도
- Comprehend Custom — 자체 분류 / 개체 학습

## 4. 관련

- [[../ml|↑ ML & AI]]
- [[../bedrock/bedrock]]
