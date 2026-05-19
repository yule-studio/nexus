---
title: "AWS Security (Hub)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:09:00+09:00
updated_at: 2026-05-19T11:00:00+09:00
tags:
  - aws
  - security
  - hub
---

# AWS Security (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 카테고리 hub |
| v.2.0.0 | 2026-05-19 | engineering-agent/tech-lead | IAM 을 별도 top-level 카테고리로 분리 ([[../iam/iam]]). 본 hub 는 KMS / Secrets Manager / WAF / Shield / GuardDuty 등 protection 도구만 |

**[[../cloud-aws|↑ AWS]]**

> IAM (identity / access) 은 별도 top-level 카테고리 [[../iam/iam]]. 본 hub 는 **데이터·트래픽 보호 도구** 만 다룬다.

---

## 1. 서비스

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **KMS** | [[kms]] | Key Management — 암호화 키 |
| **Secrets Manager** | [[secrets-manager]] | DB password / API token 안전 보관 + rotation |

---

## 2. 보안 원칙

1. **Least Privilege** — 필요한 최소 권한만 (IAM — [[../iam/iam]])
2. **Defense in Depth** — IAM + Security Group + NACL + WAF
3. **Encryption at Rest + In Transit** — KMS + TLS
4. **Audit** — CloudTrail (모든 API 호출 log)
5. **MFA** — root + admin 사용자 필수
6. **No Shared Credentials** — User / Role 각자

---

## 3. 추가 서비스 (별도 노트 예정)

- **WAF** — Web ACL (SQL injection / XSS / rate limit)
- **Shield** — DDoS
- **GuardDuty** — 이상 탐지
- **Macie** — S3 민감 데이터 검출
- **Cognito** — 사용자 인증 (login / OAuth)
- **ACM** — TLS 인증서 (무료)
- **Inspector** — 취약점 스캔

→ 각각 서비스 서브폴더 (`security/waf/waf.md` 등) 로 추가 예정.

---

## 4. 관련

- [[../cloud-aws|↑ AWS]]
- [[../iam/iam|↗ IAM (별도 카테고리)]]
- [[../../../computer-science/security-theory/security-theory|↗ 보안 이론]]
- [[../../security-ops/security-ops]]
