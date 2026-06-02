---
title: "AWS Migration Hub"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:20:00+09:00
tags:
  - aws
  - migration
---

# AWS Migration Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Migration Hub hub |

**[[../migration|↑ Migration]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

여러 마이그레이션 서비스 (MGN / DMS / DataSync / Discovery) 의 **진행 상황 통합 대시보드** + 전략 권장. 대규모 마이그레이션 프로젝트 추적.

## 2. 왜 / 언제

- 수십~수백 서버 동시 이전 추적
- 전략 권장 (7Rs — rehost / replatform / ...)
- portfolio assessment (응용 / 서버 / DB 인벤토리)

## 3. 함정

- 작은 이전 (수 개 서버) 에는 과함
- Discovery Service 와 함께 쓸 때 진가 발휘

## 4. 관련

- [[../migration|↑ Migration]]
- [[../mgn/mgn]] / [[../../database/dms/dms]] / [[../../storage/datasync/datasync]]
- [[../application-discovery/application-discovery]]
