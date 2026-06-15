---
title: "AWS Directory Service"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:29:00+09:00
tags:
  - aws
  - security
  - directory
  - active-directory
---

# AWS Directory Service

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Directory Service hub |

**[[../security|↑ Security]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **Active Directory**. Windows 워크로드 / RDS for SQL Server / WorkSpaces / FSx for Windows 의 인증 기반.

## 2. 종류

| 종류 | 의미 |
| --- | --- |
| **Managed Microsoft AD (AWS Managed AD)** | 진짜 AD (Schema 확장 가능) |
| **AD Connector** | 온프레 AD 의 proxy (AWS 안에 디렉터리 자체는 X) |
| **Simple AD** | Samba 기반 mini AD (소규모) |

## 3. 함정 / 비용

- 시간당 $0.15~0.50 (Standard / Enterprise)
- Windows / FSx / WorkSpaces 가 필요로 함
- 직원 SSO 는 **IAM Identity Center** ([[../../iam/identity-center/identity-center]]) — Directory Service 의 일반화

## 4. 관련

- [[../security|↑ Security]]
- [[../../iam/identity-center/identity-center]]
- [[../../business-apps/business-apps]] — WorkSpaces / WorkMail
- [[../../storage/fsx/fsx]] — FSx for Windows
