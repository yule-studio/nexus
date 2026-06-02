---
title: "AWS Migration & Transfer (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:00:00+09:00
tags:
  - aws
  - migration
  - hub
---

# AWS Migration & Transfer (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | 카테고리 hub |

**[[../cloud-aws|↑ AWS]]**

> 온프레 / 타 클라우드 → AWS 이전 도구. DB 마이그레이션은 [[../database/dms/dms|DMS]] (Database 카테고리에도 등장).

---

## 1. 서비스

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Migration Hub** | [[migration-hub]] | 마이그레이션 진행 상황 통합 대시보드 |
| **MGN (Application Migration Service)** | [[mgn]] | VM / 물리 서버 → EC2 lift-and-shift |
| **DMS** | [[../database/dms/dms]] | DB → DB (Database 카테고리) |
| **DataSync** | [[../storage/datasync/datasync]] | 파일 / 객체 데이터 이전 |
| **Transfer Family** | [[transfer-family]] | 매니지드 SFTP / FTPS / FTP / AS2 |
| **Snowball / Snowmobile** | [[snowball]] | 물리 디바이스로 페타바이트 이전 |
| **Application Discovery Service** | [[application-discovery]] | 온프레 인벤토리 자동 수집 |

---

## 2. 7Rs 전략

| 전략 | 의미 |
| --- | --- |
| **Retire** | 더 안 쓰는 응용 — 제거 |
| **Retain** | 그대로 둠 (당장 안 옮김) |
| **Rehost** (Lift & Shift) | 그대로 EC2 로 — MGN |
| **Replatform** | 약간 수정 (RDS 로 이전) |
| **Repurchase** | SaaS 로 교체 |
| **Refactor** | 클라우드 native 로 재작성 |
| **Relocate** | VMware → VMware Cloud on AWS |

→ 처음엔 **rehost** 로 빨리 옮긴 뒤 점진 replatform / refactor.

---

## 3. 관련

- [[../cloud-aws|↑ AWS]]
- [[../database/dms/dms]]
- [[../storage/datasync/datasync]]
- [[../storage/backup/backup]]
