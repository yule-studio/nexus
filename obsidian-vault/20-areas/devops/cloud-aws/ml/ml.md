---
title: "AWS ML & AI (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T10:32:00+09:00
tags:
  - aws
  - ml
  - ai
  - llm
  - hub
---

# AWS ML & AI (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | 카테고리 hub |

**[[../cloud-aws|↑ AWS]]**

> AWS 의 ML / AI 서비스. 자체 모델 학습 (SageMaker) vs 매니지드 LLM (Bedrock) vs 응용 API (Comprehend / Rekognition 등).

---

## 1. 서비스

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Bedrock** | [[bedrock]] | 파운데이션 모델 (Claude / Llama / Titan) API |
| **SageMaker** | (★★☆, 별도 예정) | ML 학습 / 배포 풀스택 |
| **Textract / Rekognition / Transcribe / Polly / Translate / Lex / Kendra / Personalize / Comprehend / Forecast / Fraud Detector** | (★☆☆, 별도 예정) | 응용 AI API (OCR / 이미지 / STT / TTS / NMT / 챗봇 / 검색 / 추천 / 텍스트 / 예측 / 사기) |

---

## 2. 선택

| 시나리오 | 추천 |
| --- | --- |
| **LLM 응용 (챗봇 / RAG / 요약)** | Bedrock (Claude / Llama) |
| 자체 모델 학습 / fine-tuning | SageMaker |
| OCR + 표 추출 | Textract |
| 이미지 분류 / 얼굴 / 모더레이션 | Rekognition |
| 음성 ↔ 텍스트 | Transcribe / Polly |
| 검색 (semantic) | Kendra 또는 OpenSearch + Bedrock embedding |
| 추천 | Personalize 또는 자체 (SageMaker) |

→ 신규 응용은 **LLM 기반 (Bedrock + RAG)** 이 디폴트가 됐다. 응용별 API 서비스 (Comprehend / Personalize 등) 는 점차 LLM 으로 대체 추세.

---

## 3. LLM 응용 표준 구성

```
응용 → API Gateway → Lambda → Bedrock (Claude)
                        ↓
                 OpenSearch (vector) ← S3 (문서)
                        ↓
                 DynamoDB (대화 이력)
```

자세히 → [[bedrock]]

---

## 4. 관련

- [[../cloud-aws|↑ AWS]]
- [[../../../ai-agent/ai-agent|↗ AI Agent 영역]]
- [[../../mlops/mlops|↗ MLOps]]
