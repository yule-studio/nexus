---
title: "AWS Lake Formation"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:02:00+09:00
tags:
  - aws
  - analytics
  - data-lake
  - lake-formation
---

# AWS Lake Formation

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Lake Formation hub |

**[[../analytics|↑ Analytics]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

S3 + Glue Data Catalog 위의 **데이터 레이크 권한 / 거버넌스 통합**. row/column-level 보안 + cross-account 공유.

## 2. 왜 / 언제

- 데이터 레이크의 정밀 권한 (부서별 row 마스킹)
- Glue 카탈로그의 access 통합 관리
- 멀티 account 데이터 공유 (LF tag 기반)
- Athena / Redshift Spectrum / EMR 모두 동일 권한 모델

## 3. 함정

- IAM + Lake Formation 둘 다 권한 — 디버그 복잡
- 작은 데이터 레이크엔 과한 거버넌스
- LF tag 가 권한 관리의 핵심 (자원별 ARN 보다 확장 친화)

## 4. 관련

- [[../analytics|↑ Analytics]]
- [[../glue/glue]] / [[../athena/athena]]
- [[../../storage/s3/s3]]
