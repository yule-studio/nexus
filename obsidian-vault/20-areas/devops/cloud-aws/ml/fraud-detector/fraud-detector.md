---
title: "Amazon Fraud Detector"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:19:00+09:00
tags:
  - aws
  - ml
  - fraud
---

# Amazon Fraud Detector

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Fraud Detector hub |

**[[../ml|↑ ML & AI]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

ML 기반 **사기 거래 탐지** — 결제 사기 / 가짜 계정 / 프로모션 어뷰징. Amazon.com 의 fraud detection 모델 재사용.

## 2. 왜 / 언제

- 신용카드 / 결제 사기 탐지
- 가짜 회원가입 / 프로모션 abuse
- 자체 ML 팀 없이 빠른 도입

## 3. 함정 / 대안

- 학습 데이터 (라벨링된 사기 사례) 필요
- 신규 응용 = **SageMaker / Bedrock 자체 모델** 더 유연
- GuardDuty 는 AWS 자원 사기, Fraud Detector 는 응용 비즈니스 사기

## 4. 관련

- [[../ml|↑ ML & AI]]
- [[../sagemaker/sagemaker]]
- [[../../security/guardduty/guardduty]]
