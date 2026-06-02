---
title: "AWS DataZone"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:03:00+09:00
tags:
  - aws
  - analytics
  - data-catalog
  - datazone
---

# AWS DataZone

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | DataZone hub |

**[[../analytics|↑ Analytics]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

데이터 **카탈로그 + 검색 + 공유 / 거버넌스 포털**. 조직 단위로 "어떤 데이터 셋이 어디 있나" 검색 + 요청 / 승인 워크플로우.

## 2. 왜 / 언제

- 대형 조직 — 데이터 셋이 분산되어 검색 어려움
- data producer / consumer 분리 (셀프 서비스)
- 비즈니스 용어 사전 / 메타 (data dictionary)
- Lake Formation 권한과 통합

## 3. 함정

- 2023+ 신규 서비스 — 생태계 미숙
- 대안: **Atlan / Collibra / Alation** SaaS
- 작은 조직엔 과한 도구

## 4. 관련

- [[../analytics|↑ Analytics]]
- [[../lake-formation/lake-formation]]
- [[../glue/glue]]
