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

### 1.1 플랫폼

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **SageMaker** | [[sagemaker/sagemaker\|sagemaker]] | 매니지드 ML 학습 / 튜닝 / 배포 / MLOps 풀스택 — notebook → training → endpoint → monitor |

### 1.2 언어·텍스트

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Comprehend** | [[comprehend/comprehend\|comprehend]] | NLP API — 감정 / 개체 / 키 구문 / 언어 / 분류 / 토픽 모델링 + PII 검출 |
| **Translate** | [[translate/translate\|translate]] | 기계 번역 (NMT) — 75+ 언어, real-time + batch + custom terminology |
| **Textract** | [[textract/textract\|textract]] | OCR + 표 / 양식 / 영수증 / ID 카드 구조 추출 — 단순 OCR 보다 구조 인식 강함 |
| **Transcribe** | [[transcribe/transcribe\|transcribe]] | 음성 → 텍스트 (STT) — batch / streaming / Medical / Call Analytics, 한국어 포함 |
| **Polly** | [[polly/polly\|polly]] | 텍스트 → 음성 (TTS) — 60+ 언어, Neural / Generative voice 고품질 |

### 1.3 비전

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Rekognition** | [[rekognition/rekognition\|rekognition]] | 이미지 / 영상 인식 API — 객체 / 얼굴 / 모더레이션 / 텍스트, Custom Labels 자체 모델 |

### 1.4 검색·추천·예측

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Kendra** | [[kendra/kendra\|kendra]] | 기업용 의미 검색 (semantic search) — S3 / SharePoint / Confluence 등 커넥터 + LLM 답변 추출 |
| **Personalize** | [[personalize/personalize\|personalize]] | 매니지드 추천 시스템 — 사용자 활동 / 아이템 메타 → 개인화 추천 / 유사 아이템 / 트렌딩 |
| **Forecast** | [[forecast/forecast\|forecast]] | 시계열 예측 (수요 / 매출 / 트래픽) — 신규 가입 닫힘 (2024, 기존 사용자만) |
| **Fraud Detector** | [[fraud-detector/fraud-detector\|fraud-detector]] | ML 기반 사기 거래 탐지 — 결제 사기 / 가짜 계정 / 프로모션 어뷰징 |

### 1.5 대화

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Lex** | [[lex/lex\|lex]] | 챗봇 / 음성 봇 (Alexa 엔진) — intent + slot + fulfillment Lambda 모델, Connect 통합 |

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
