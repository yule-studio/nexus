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

### 1.1 암호·키

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **ACM** | [[acm/acm\|acm]] | 무료 TLS 인증서 발급 + 자동 갱신 — ALB / CloudFront / API Gateway 의 HTTPS |
| **CloudHSM** | [[cloudhsm/cloudhsm\|cloudhsm]] | FIPS 140-2 Level 3 전용 HSM — KMS 보다 강한 키 격리 (금융 / 의료 / 정부 규제용) |

### 1.2 인증·디렉터리

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Cognito** | [[cognito/cognito\|cognito]] | 응용 사용자의 로그인 / 회원가입 / 소셜 로그인 / MFA 를 매니지드로 제공 |
| **Directory Service** | [[directory-service/directory-service\|directory-service]] | 매니지드 Active Directory — Windows 워크로드 / RDS / WorkSpaces / FSx 인증 기반 |

### 1.3 위협탐지·태세

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **GuardDuty** | [[guardduty/guardduty\|guardduty]] | CloudTrail / VPC Flow Logs / DNS Logs 를 ML 분석 — 이상 행동 / 위협 자동 탐지 |
| **Inspector** | [[inspector/inspector\|inspector]] | EC2 / Lambda / 컨테이너 이미지의 CVE 취약점 + 네트워크 노출 자동 스캔 |
| **Macie** | [[macie/macie\|macie]] | S3 안의 민감 데이터 (PII / 신용카드 / 주민번호) 자동 검출 + 분류 |
| **Security Hub** | [[security-hub/security-hub\|security-hub]] | GuardDuty / Inspector / Macie finding 단일 view 통합 + 컴플라이언스 점검 |

### 1.4 경계·차단

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **WAF** | [[waf/waf\|waf]] | L7 HTTP 방화벽 — SQLi / XSS / bot / rate limit / 국가 차단 룰을 ALB / CloudFront 에 배치 |
| **Shield** | [[shield/shield\|shield]] | DDoS 공격 방어 (Standard 무료, Advanced 월 $3000~) |
| **Firewall Manager** | [[firewall-manager/firewall-manager\|firewall-manager]] | Organizations 단위로 WAF / Shield / Security Group / Network Firewall 룰 일괄 관리 |

### 1.5 거버넌스

| 서비스 | 노트 | 한 줄 |
| --- | --- | --- |
| **Artifact** | [[artifact/artifact\|artifact]] | AWS 컴플라이언스 보고서 (SOC / ISO / PCI / HIPAA) 다운로드 + NDA / BAA 계약 |
| **Parameter Store** | [[parameter-store/parameter-store\|parameter-store]] | 응용 설정값 + secret 을 계층적 경로로 저장하는 매니지드 KV (KMS 통합) |

---

## 2. 보안 원칙

1. **Least Privilege** — 필요한 최소 권한만 (IAM — [[../iam/iam]])
2. **Defense in Depth** — IAM + Security Group + NACL + WAF
3. **Encryption at Rest + In Transit** — KMS + TLS
4. **Audit** — CloudTrail (모든 API 호출 log)
5. **MFA** — root + admin 사용자 필수
6. **No Shared Credentials** — User / Role 각자

---

## 3. 관련

- [[../cloud-aws|↑ AWS]]
- [[../iam/iam|↗ IAM (별도 카테고리)]]
- [[../../../computer-science/security-theory/security-theory|↗ 보안 이론]]
- [[../../security-ops/security-ops]]
