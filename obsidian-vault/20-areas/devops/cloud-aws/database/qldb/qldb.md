---
title: "Amazon QLDB (Quantum Ledger Database)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:51:00+09:00
tags:
  - aws
  - database
  - ledger
  - qldb
  - deprecated
---

# Amazon QLDB

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | QLDB hub (종료 발표) |

**[[../database|↑ Database]]** · **[[../../cloud-aws|↑↑ AWS]]**

> **종료 (2025-07-31)** — 신규 가입 닫힘. Aurora PostgreSQL audit log + 자체 hash chain 또는 [[../../management/organizations/organizations|Org]] + CloudTrail 로 이전.

---

## 1. 한 줄

변경 불가 원장 (immutable ledger) DB. **종료** — Aurora 의 system-versioned 테이블 / 자체 cryptographic verification 으로 대체.

## 2. 후속

- 응용 audit ledger = Aurora PostgreSQL + history 테이블 + hash chain
- 시스템 audit = [[../../observability/cloudtrail/cloudtrail|CloudTrail]] + S3 Object Lock

## 3. 관련

- [[../database|↑ Database]]
- [[../aurora/aurora]]
- [[../../observability/cloudtrail/cloudtrail]]
