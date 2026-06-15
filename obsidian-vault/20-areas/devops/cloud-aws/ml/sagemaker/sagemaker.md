---
title: "Amazon SageMaker"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:55:00+09:00
tags:
  - aws
  - ml
  - sagemaker
  - mlops
---

# Amazon SageMaker

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | SageMaker hub |

**[[../ml|↑ ML & AI]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **ML 학습 / 튜닝 / 배포 / MLOps 풀스택**. notebook → training → tuning → endpoint → monitor → registry. LLM 응용은 [[../bedrock/bedrock|Bedrock]], 자체 모델 학습은 SageMaker.

## 2. 왜 / 언제

- 자체 모델 학습 (vision / 추천 / time-series)
- 오픈소스 LLM fine-tuning (Llama / Mistral)
- ML 추론 endpoint 배포 (실시간 / 배치 / serverless)
- 데이터 사이언티스트의 표준 워크벤치
- 분산 GPU 학습 (멀티 노드)

## 3. 주요 서브 제품

| 서브 | 의미 |
| --- | --- |
| **Studio** | 통합 IDE (Notebook + Pipeline + Model Registry) |
| **Notebooks** | 매니지드 Jupyter |
| **Training Jobs** | 분산 학습 (Spot / GPU) |
| **HyperPod** | 대규모 fault-tolerant 학습 |
| **Inference Endpoints** | 실시간 / 비동기 / 서버리스 / 배치 |
| **Model Registry** | 모델 버전 / 승인 / 배포 |
| **Pipelines** | ML CI/CD (학습 → 평가 → 배포) |
| **Feature Store** | online / offline feature |
| **Ground Truth** | 라벨링 (인간 / 자동) |
| **Clarify / Model Monitor** | 편향 / drift |
| **JumpStart** | 사전 학습 모델 / 솔루션 즉시 사용 |

## 4. 표준 응용 ML 흐름

```
Raw data (S3)
   ↓ Glue / Processing Job (feature)
Feature Store
   ↓ Training Job (Spot GPU)
Model Registry (Approved)
   ↓ Pipeline 트리거
Endpoint (멀티 모델 가능)
   ↓ 응용 호출
Model Monitor (drift / 편향)
```

## 5. 비용 / 함정

- 학습 비용 — 인스턴스 시간 (GPU $1-30/시간) — Spot 으로 70% 절감 가능
- endpoint 는 **항상 켜져 있음** — 트래픽 적으면 Serverless / 비동기 endpoint 검토
- Studio 도 인스턴스 시간 청구 — 사용 안 할 때 stop
- LLM 사용 = **Bedrock 이 훨씬 단순** (학습 안 할 거면)
- multi-model endpoint 로 모델 여러 개를 한 인스턴스에서

## 6. 관련

- [[../ml|↑ ML & AI]]
- [[../bedrock/bedrock]] — LLM 호출
- [[../textract/textract]] — OCR
- [[../../../ai/ai|↗ AI 영역]]
- [[../../../devops/mlops/mlops|↗ MLOps]]
