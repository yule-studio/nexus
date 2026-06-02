---
title: "Amazon Rekognition"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:11:00+09:00
tags:
  - aws
  - ml
  - vision
---

# Amazon Rekognition

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Rekognition hub |

**[[../ml|↑ ML & AI]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

이미지 / 영상 인식 API — **객체 / 얼굴 / 얼굴 비교 / 모더레이션 / 텍스트 (간단 OCR) / 라벨**. Custom Labels 로 자체 모델도.

## 2. 왜 / 언제

- 콘텐츠 모더레이션 (성인 / 폭력)
- 얼굴 비교 (신원 확인 — Liveness 포함)
- 응용 사진 자동 태깅
- 영상의 객체 / 활동 인식
- OCR 은 [[../textract/textract|Textract]] 가 더 강함

## 3. 함정

- 얼굴 인식 = 법적 / 윤리 검토 필요 (EU AI Act 등)
- 한국어 텍스트 정확도 한정 — Textract / Bedrock 권장
- 단가 image 당 (~$0.001) — 대량 호출 시 누적

## 4. 관련

- [[../ml|↑ ML & AI]]
- [[../textract/textract]]
- [[../bedrock/bedrock]]
