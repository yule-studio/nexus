---
title: "AWS IAM Identity Center (구 AWS SSO)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T10:42:00+09:00
tags:
  - aws
  - iam
  - sso
  - identity
---

# AWS IAM Identity Center

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Identity Center hub |

**[[../iam|↑ IAM]]** · **[[../../cloud-aws|↑↑ AWS]]**

> 옛 이름: **AWS SSO** (2022 rebrand).

---

## 1. 한 줄

여러 AWS account / 응용 / 외부 IdP (Okta / Entra / Google) 를 **하나의 로그인** 으로 묶는 매니지드 SSO. **IAM user 의 대체.**

---

## 2. 왜 / 언제

- 직원 / 외주에게 IAM user (access key) 발급 X → SSO + 임시 자격
- 여러 account (prod / staging / dev / sandbox) 를 한 번 로그인으로 전환
- MFA / 세션 만료 / 권한 부여를 **중앙 관리**
- 외부 IdP (Okta / Entra ID) 연동 (offboarding 자동화)
- AWS CLI / SDK 도 `aws sso login` 으로 임시 token

---

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Identity source** | 내장 디렉터리 / AD / 외부 IdP (Okta 등) |
| **User / Group** | 사람 / 그룹 |
| **Permission Set** | 어떤 권한 (IAM policy 묶음) |
| **Account assignment** | `(User/Group) × Account × Permission Set` |
| **AWS access portal** | `https://<id>.awsapps.com/start` — 사용자 진입점 |

---

## 4. 표준 패턴

```
Okta (외부 IdP)
   ↓ SAML
IAM Identity Center
   ↓
┌──────────────────┐
│ prod account     │ ← "AdministratorAccess" Permission Set
│ staging account  │ ← "PowerUserAccess"
│ dev account      │ ← "AdministratorAccess"
└──────────────────┘
   ↓ CLI
aws sso login --profile prod
aws s3 ls --profile prod
```

`~/.aws/config`:
```ini
[profile prod]
sso_start_url = https://myorg.awsapps.com/start
sso_region = us-east-1
sso_account_id = 111122223333
sso_role_name = AdministratorAccess
region = ap-northeast-2
```

---

## 5. IAM user vs Identity Center 비교

| | IAM user | Identity Center |
| --- | --- | --- |
| 자격 | 영구 access key | **임시 STS token** |
| 다중 account | account 마다 user | **1 user → N account** |
| 외부 IdP | X | ✅ |
| MFA | 가능 (수동) | **강제 / 외부 IdP** |
| 권장 | 응용 / 자동화 X | **사람 / CLI / 콘솔** |

→ 신규 조직 = **Identity Center 만**. IAM user 는 거의 안 만든다 (Role 사용).

---

## 6. 함정

- **Permission Set 너무 광범위** — `AdministratorAccess` 남발 금지, 역할별 customer-managed Permission Set.
- **세션 시간** — 기본 1시간. 길게는 12시간까지 (응용 호환성 확인).
- **외부 IdP attribute mapping** — 그룹 이름 일치 확인.
- **Organizations 필요** — Identity Center 는 Organizations 위에서만 동작.

---

## 7. 관련

- [[../iam|↑ IAM]]
- [[../../management/organizations/organizations]]
- [[../../security/security]]
