---
title: "AWS Storage Gateway"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:16:00+09:00
tags:
  - aws
  - storage
  - hybrid
---

# AWS Storage Gateway

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Storage Gateway hub |

**[[../storage|↑ Storage]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

온프레미스 ↔ AWS 스토리지 **하이브리드 게이트웨이**. 온프레 환경에서 S3 / EBS 를 로컬 NFS / SMB / iSCSI 처럼 보이게.

## 2. 왜 / 언제

| 종류 | 의미 |
| --- | --- |
| **S3 File Gateway** | NFS / SMB → S3 (가장 자주) |
| **FSx File Gateway** | SMB → FSx for Windows |
| **Volume Gateway** | iSCSI → EBS (백업) |
| **Tape Gateway** | 가상 tape → S3 Glacier (백업 시스템 이식) |

→ 클라우드 마이그레이션 중간 단계 / 백업 용도.

## 3. 함정 / 비용

- 게이트웨이 VM (VMware / Hyper-V / EC2) 설치 필요
- 로컬 cache 디스크 — 자주 쓰는 데이터 캐싱
- 풀 마이그레이션 후엔 게이트웨이 제거 (단순 S3 직접 사용)

## 4. 관련

- [[../storage|↑ Storage]]
- [[../s3/s3]]
- [[../datasync/datasync]]
- [[../../migration/migration]]
