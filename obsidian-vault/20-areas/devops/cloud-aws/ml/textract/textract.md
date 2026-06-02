---
title: "Amazon Textract"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:56:00+09:00
tags:
  - aws
  - ml
  - ocr
  - textract
---

# Amazon Textract

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Textract hub |

**[[../ml|↑ ML & AI]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

OCR + **표 / 양식 / 영수증 / ID 카드 구조 추출**. 단순 OCR (Rekognition / Tesseract) 보다 구조 인식이 강함. PDF / 이미지 모두.

## 2. 왜 / 언제

- 송장 / 영수증에서 항목 / 금액 자동 추출
- 회원 가입 신분증 / 운전면허 자동 파싱
- 의료 / 보험 양식 자동화
- 계약서에서 표 / signature 추출
- LLM (Bedrock) 의 사전 단계 — PDF → 구조화 데이터

## 3. 주요 API

| API | 의미 |
| --- | --- |
| **DetectDocumentText** | 단순 OCR (텍스트 + 좌표) |
| **AnalyzeDocument** | + 표 / 양식 / signature / query |
| **AnalyzeExpense** | 송장 / 영수증 특화 (항목 / 금액 / VAT) |
| **AnalyzeID** | 신분증 (운전면허 / 여권) 특화 |
| **AnalyzeLending** | 대출 문서 패키지 |
| **StartDocumentAnalysis** | 큰 PDF 비동기 |

## 4. 표준 패턴 — 송장 처리

```
S3 (PDF 업로드)
   ↓ EventBridge
   ↓ Lambda
   ↓ Textract.AnalyzeExpense
   ↓ 결과 JSON → DynamoDB
   ↓
응용 UI
```

→ 큰 PDF (수십 페이지) 는 `StartDocumentAnalysis` 비동기 + SNS 완료 알림.

## 5. 비용 / 함정

- 페이지 단위 ($0.0015 ~ $0.10 / page, API 종류별)
- AnalyzeExpense / AnalyzeID 는 비싸지만 정확도 우수
- LLM (Bedrock) 직접 PDF 처리 vs Textract — **혼합 권장**: Textract 가 텍스트 추출 → Bedrock 이 의미 해석
- 한글 — 일부 API 만 지원 (DetectDocumentText / AnalyzeDocument 의 한글 정확도 점차 개선)
- async 결과는 7일 후 자동 삭제

## 6. 관련

- [[../ml|↑ ML & AI]]
- [[../bedrock/bedrock]] — 후속 의미 해석
- [[../sagemaker/sagemaker]]
- [[../../storage/s3/s3]]
