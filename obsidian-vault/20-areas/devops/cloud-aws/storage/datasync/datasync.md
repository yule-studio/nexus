---
title: "AWS DataSync"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:17:00+09:00
tags:
  - aws
  - storage
  - migration
  - datasync
---

# AWS DataSync

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | DataSync hub |

**[[../storage|↑ Storage]]** · **[[../../cloud-aws|↑↑ AWS]]**

> [[../../migration/migration|Migration]] 카테고리에서도 등장.

---

## 1. 한 줄

**대용량 파일 / 객체 데이터 자동 이전**. 온프레 NFS / SMB / HDFS / 다른 클라우드 ↔ S3 / EFS / FSx. 검증 / 암호화 / 일정 자동.

## 2. 왜 / 언제

- 데이터센터 → AWS 마이그레이션 (TB ~ PB)
- 다른 클라우드 (GCS / Azure) ↔ AWS
- 정기 데이터 이중화 / 백업
- 온프레 NAS → S3 데이터 레이크 적재

## 3. 핵심 / 함정

| 개념 | 의미 |
| --- | --- |
| **Agent** | 온프레 VM (VMware / KVM / EC2) — 트래픽 가속 + 검증 |
| **Location** | source / destination (NFS / SMB / HDFS / S3 / EFS / FSx) |
| **Task** | 한 번 또는 schedule 실행 |
| **Bandwidth limit** | 업무 시간 트래픽 제한 |
| **함정** | 매우 큰 (PB) = Snowball + DataSync 조합 권장 |

## 4. 관련

- [[../storage|↑ Storage]]
- [[../storage-gateway/storage-gateway]]
- [[../../migration/migration]]
- [[../../migration/snowball/snowball]]
