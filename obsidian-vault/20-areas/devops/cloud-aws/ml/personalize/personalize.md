---
title: "Amazon Personalize"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:17:00+09:00
tags:
  - aws
  - ml
  - recommendation
---

# Amazon Personalize

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Personalize hub |

**[[../ml|↑ ML & AI]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **추천 시스템** — Amazon.com 의 노하우 기반. 사용자 활동 / 아이템 메타 → 개인화 추천 / 유사 아이템 / 트렌딩.

## 2. 왜 / 언제

- 이커머스 / 콘텐츠 추천 (영화 / 음악 / 뉴스)
- 자체 ML 팀 없이 빠른 추천
- "사용자가 산 것" / "유사 사용자" / "트렌딩" 레시피

## 3. 함정 / 대안

- 모델 학습 / hosting 비용 (월 수백 $)
- 작은 데이터 (10K interaction 미만) 는 효과 X
- 신규 = pgvector + Bedrock embedding 으로 자체 구축이 더 저렴 / 유연한 경우도

## 4. 관련

- [[../ml|↑ ML & AI]]
- [[../sagemaker/sagemaker]]
- [[../bedrock/bedrock]]
