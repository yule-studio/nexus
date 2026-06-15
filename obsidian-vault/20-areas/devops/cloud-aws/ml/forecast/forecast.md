---
title: "Amazon Forecast"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:18:00+09:00
tags:
  - aws
  - ml
  - forecast
  - timeseries
---

# Amazon Forecast

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Forecast hub |

**[[../ml|↑ ML & AI]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

시계열 예측 — 수요 / 매출 / 트래픽. ARIMA / Prophet / DeepAR+ 등 알고리즘 자동 선택. **신규 가입 닫힘 (2024)** — 기존 사용자만.

## 2. 왜 / 언제

- 수요 예측 / 매출 예측 / inventory planning
- 자체 ML 팀 없이 빠른 시계열 모델
- 신규 = **SageMaker Canvas / Bedrock + 자체 데이터** 권장

## 3. 함정

- **신규 가입 닫힘** — AWS 가 SageMaker 로 통합 방향
- 학습 데이터 최소 요구 (수천 + 시계열 길이)

## 4. 관련

- [[../ml|↑ ML & AI]]
- [[../sagemaker/sagemaker]]
- [[../../database/timestream/timestream]]
