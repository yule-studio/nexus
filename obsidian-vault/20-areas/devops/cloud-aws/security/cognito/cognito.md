---
title: "Amazon Cognito"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:10:00+09:00
tags:
  - aws
  - security
  - auth
  - cognito
---

# Amazon Cognito

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Cognito hub |

**[[../security|↑ Security]]** · **[[../../cloud-aws|↑↑ AWS]]**

> [[../../frontend/frontend|Frontend]] 카테고리에서도 등장.

---

## 1. 한 줄

응용 사용자 (end-user) 의 **로그인 / 회원가입 / 소셜 로그인 / MFA** 를 매니지드로 제공. IAM 은 AWS 자원 접근, Cognito 는 응용 사용자.

## 2. 왜 / 언제

- 응용에 자체 로그인 시스템 (Auth0 / Firebase Auth 대안)
- Google / Apple / Facebook 소셜 로그인 통합
- 모바일 / SPA 에서 안전한 토큰 발급 (JWT)
- 사용자가 임시 AWS 자격 (S3 직접 업로드 등) 을 받아야 할 때
- SAML / OIDC 외부 IdP 연동

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **User Pool** | 사용자 디렉터리 + 로그인 (JWT 발급) |
| **Identity Pool** (Federated Identities) | 외부 IdP / User Pool → **임시 AWS 자격** 부여 |
| **App Client** | User Pool 안의 응용 client (web / mobile) |
| **Hosted UI** | Cognito 가 제공하는 로그인 화면 |
| **Triggers** | Lambda (pre-signup / post-confirm / custom message 등) |
| **JWT** | ID token (인증) / Access token (인가) / Refresh token |

## 4. 표준 패턴

```
React/Mobile → Hosted UI / SDK → Cognito User Pool
                                    ↓ JWT
                              API Gateway (JWT authorizer)
                                    ↓
                                 Lambda / 응용
```

- SPA = **PKCE flow** (`aws-amplify` 또는 `oidc-client` 라이브러리)
- API 보호 = API Gateway **Cognito Authorizer** (코드 0줄)
- S3 직접 업로드 = Identity Pool → 임시 자격

## 5. 비용 / 함정

- 50,000 MAU 까지 무료, 이후 MAU 당 과금
- **token 검증** 응용 측 캐싱 (JWKs 자주 호출 X)
- **Hosted UI 커스텀** 한정 — 풀 커스텀하려면 자체 UI + API
- **email 발송** SES 사용 권장 (기본은 일일 50 통 제한)
- **password policy / MFA** 변경 후 기존 사용자 영향 확인

## 6. 관련

- [[../security|↑ Security]]
- [[../../iam/identity-center/identity-center]] — 직원용 SSO (Cognito 는 end-user)
- [[../../frontend/frontend]]
- [[../../network/api-gateway/api-gateway]]
