---
title: "AWS Transfer Family"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:22:00+09:00
tags:
  - aws
  - migration
  - sftp
  - transfer-family
---

# AWS Transfer Family

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Transfer Family hub |

**[[../migration|↑ Migration]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **SFTP / FTPS / FTP / AS2** endpoint — S3 / EFS 로 직접 적재. 외부 파트너의 파일 업로드 수용.

## 2. 왜 / 언제

- 거래처 / 은행 / 정부의 SFTP 업로드 인터페이스
- B2B EDI (AS2)
- 자체 SFTP 서버 운영 부담 회피
- Workflows — 업로드 후 자동 후처리 (Lambda / Step Functions)

## 3. 함정

- endpoint 시간당 ~$0.30/시간 (월 $200+) + 데이터 비용
- 트래픽 적으면 EC2 + OpenSSH 가 훨씬 쌈
- IAM 또는 외부 IdP (LDAP / Custom Lambda) 인증

## 4. 관련

- [[../migration|↑ Migration]]
- [[../../storage/s3/s3]] / [[../../storage/efs/efs]]
- [[../../app-integration/step-functions/step-functions]]
