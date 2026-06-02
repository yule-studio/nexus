---
title: "Amazon DocumentDB"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:21:00+09:00
tags:
  - aws
  - database
  - nosql
  - mongodb
---

# Amazon DocumentDB

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | DocumentDB hub |

**[[../database|↑ Database]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**MongoDB 호환** (API 수준) 매니지드 document DB. Aurora 처럼 분산 스토리지 위. 기존 MongoDB 응용을 AWS 로 옮길 때 candidate.

## 2. 왜 / 언제

- 기존 MongoDB 응용을 AWS 매니지드로
- JSON document 모델 + ACID 트랜잭션
- 자체 MongoDB 운영 부담 회피

## 3. 함정 / 비교

- **MongoDB 호환 ≠ 100% MongoDB** — 일부 API / aggregation 차이
- 최신 MongoDB 기능은 늦게 / 일부 미지원
- 비용도 적지 않음 (Aurora 와 비슷)
- 신규 응용 = **DynamoDB** (AWS native, 더 저렴 / 확장)
- MongoDB Atlas on AWS 가 정합성 / 기능 더 좋을 수도

## 4. 관련

- [[../database|↑ Database]]
- [[../dynamodb/dynamodb]]
- [[../aurora/aurora]]
