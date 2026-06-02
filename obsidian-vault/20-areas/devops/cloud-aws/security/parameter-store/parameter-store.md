---
title: "AWS Systems Manager Parameter Store"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T10:43:00+09:00
tags:
  - aws
  - security
  - secrets
  - configuration
  - ssm
---

# AWS Systems Manager Parameter Store

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Parameter Store hub |

**[[../security|↑ Security]]** · **[[../../cloud-aws|↑↑ AWS]]**

> Systems Manager ([[../../management/ssm/ssm]]) 의 일부이지만, **응용 설정 / secret** 측면에서 보안 카테고리에 함께 둠.

---

## 1. 한 줄

응용 **설정값 + secret** 을 계층적 경로 (`/app/prod/db/password`) 로 저장하는 매니지드 KV. KMS 통합 + IAM 권한 + 버전 관리.

---

## 2. 왜 / 언제

- 코드 / 환경변수 / `.env` 에 secret 두지 않기
- 환경별 (`dev / staging / prod`) 설정 분리
- 응용이 시작할 때 IAM role 로 가져오기
- 비밀번호 / API key / 설정값 통합 관리

---

## 3. Parameter Store vs Secrets Manager

| | Parameter Store | Secrets Manager |
| --- | --- | --- |
| 기본 가격 | **무료** (Standard) | **$0.40 / secret / 월** + API 호출 |
| 회전 (Rotation) | X (Lambda 직접) | **자동** (RDS / Redshift / DocumentDB 등) |
| Cross-region 복제 | X | ✅ |
| 크기 | 4KB (Standard) / 8KB (Advanced) | 64KB |
| 버전 | 100 | 무제한 |
| 사용 빈도 | **일반 설정 + 가벼운 secret** | **DB password / 자동 회전 필요한 secret** |

→ 가벼우면 Parameter Store. 회전 필요하면 Secrets Manager. ([[../secrets-manager/secrets-manager]])

---

## 4. 타입

| 타입 | 의미 |
| --- | --- |
| **String** | 평문 |
| **StringList** | comma 구분 (`a,b,c`) |
| **SecureString** | KMS 로 암호화 (secret 권장) |

Tier:
- **Standard** — 무료, 4KB, 10000개/account
- **Advanced** — 유료, 8KB, 정책 / 만료 / 더 많은 개수

---

## 5. 표준 패턴

### 5.1 계층적 네이밍

```
/myapp/prod/db/host
/myapp/prod/db/password   (SecureString)
/myapp/prod/redis/url
/myapp/staging/db/...
```

→ 응용은 `GetParametersByPath /myapp/prod/` 로 한 번에 다 가져옴.

### 5.2 응용 시작 시 로드

```python
import boto3
ssm = boto3.client("ssm")
resp = ssm.get_parameters_by_path(
    Path="/myapp/prod/", Recursive=True, WithDecryption=True
)
config = {p["Name"]: p["Value"] for p in resp["Parameters"]}
```

### 5.3 ECS / Lambda 환경변수 직접 주입

```json
{
  "secrets": [
    { "name": "DB_PASSWORD",
      "valueFrom": "arn:aws:ssm:...:parameter/myapp/prod/db/password" }
  ]
}
```

→ Task definition / Lambda 가 자동으로 fetch + decrypt.

---

## 6. 함정 / 비용

- **SecureString** 의 KMS 호출 — 호출당 과금. caching 권장 (AWS SDK 자동 cache 없음).
- **Throttling** — API rate limit. 응용 시작 시 한 번만 fetch.
- **Cross-account** — Resource Policy 설정.
- **버전** — 100 개 넘으면 오래된 것 자동 삭제.
- **변경 감지** — EventBridge `Parameter Store Change` 이벤트 → 응용 reload.

---

## 7. 관련

- [[../security|↑ Security]]
- [[../secrets-manager/secrets-manager]]
- [[../../management/ssm/ssm]]
- [[../../iam/iam]]
