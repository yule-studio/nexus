---
title: "AWS Backup"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:52:00+09:00
tags:
  - aws
  - storage
  - backup
  - dr
---

# AWS Backup

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Backup hub |

**[[../storage|↑ Storage]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

EBS / RDS / Aurora / DynamoDB / EFS / FSx / S3 / EC2 / Storage Gateway 등 **다양한 자원의 백업을 일원화** + 보존 / 복제 / 격리 / 규정 준수.

## 2. 왜 / 언제

- 여러 서비스의 backup 정책 한 곳 (tag 기반)
- 규정 (GDPR / HIPAA / KISA) 요구 보존 / 격리
- ransomware 대비 — Vault Lock (immutable)
- DR (Disaster Recovery) — cross-region / cross-account 복제
- 백업 컴플라이언스 보고 (Backup Audit Manager)

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Backup Plan** | 일정 + 보존 + 복제 룰 묶음 |
| **Backup Vault** | 백업 저장 위치 (KMS 키별 격리) |
| **Vault Lock** | WORM 모드 — 보존 기간 안 변경 / 삭제 불가 (compliance / governance 모드) |
| **Resource Assignment** | 백업 대상 (tag 기반 또는 ARN) |
| **Recovery Point** | 한 시점 스냅샷 |
| **Backup Audit Manager** | 컴플라이언스 룰 (예: "모든 RDS 가 주 1회 백업") |
| **Cross-account / region** | 다른 account / region 으로 복제 |

## 4. 표준 패턴 — Tag 기반 + Vault Lock

```yaml
Plan:
  rules:
    - schedule: cron(0 18 * * ? *)        # 매일 18:00 UTC
      target_vault: prod-vault
      lifecycle:
        cold_after_days: 30
        delete_after_days: 365
      copy_to:
        - account: log-archive
          region: us-west-2

Resource Selection:
  tags:
    - Backup = true

Vault:
  encryption: KMS
  lock:
    mode: COMPLIANCE     # 변경 / 삭제 절대 불가 (만료 전)
    min_retention: 30
    max_retention: 365
```

## 5. 비용 / 함정

- 자원별 GB 단위 + 복제 비용 (cross-region 비쌈)
- **Warm vs Cold tier** — 30일 후 cold 로 (싸짐, 복원 시간 ↑)
- **Vault Lock COMPLIANCE 모드는 진짜 불가** — 잘못 잠그면 root 도 못 풉. governance 모드로 시작 권장
- **EBS 스냅샷 vs Backup** — Backup 이 더 비싸지만 정책 일관성
- 자원 종류별 백업 옵션 (예: RDS 의 PITR, Aurora 의 backtrack) 와 별개

## 6. 관련

- [[../storage|↑ Storage]]
- [[../ebs/ebs]]
- [[../../database/rds/rds]]
- [[../../security/kms/kms]]
