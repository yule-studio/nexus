---
title: "Amazon Lex"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:15:00+09:00
tags:
  - aws
  - ml
  - chatbot
  - lex
---

# Amazon Lex

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Lex hub |

**[[../ml|↑ ML & AI]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

챗봇 / 음성 봇 — Alexa 엔진. **intent + slot + fulfillment Lambda** 모델. Connect (콜센터) 와 자주 통합.

## 2. 왜 / 언제

- 콜센터 IVR (Amazon Connect 와 통합)
- 응용 안 챗봇 (FAQ / 주문)
- 음성 / 텍스트 둘 다
- 정형화된 intent 가 명확한 워크플로우

## 3. 함정 / 대안

- 신규 챗봇 = **Bedrock Claude + tool use** 가 훨씬 유연
- Lex 는 학습 데이터 + intent 설계 필요 (정형 / 예측 가능 워크플로우에 적합)
- 자유로운 대화 / 추론 = LLM. 결정 트리 / 주문 흐름 = Lex

## 4. 관련

- [[../ml|↑ ML & AI]]
- [[../bedrock/bedrock]]
- [[../../business-apps/business-apps]] — Connect
