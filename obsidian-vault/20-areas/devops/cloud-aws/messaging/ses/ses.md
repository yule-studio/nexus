---
title: "Amazon SES (Simple Email Service)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T10:51:00+09:00
tags:
  - aws
  - messaging
  - email
  - ses
---

# Amazon SES

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | SES hub |

**[[../messaging|↑ Messaging]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **이메일 발송 / 수신** 서비스. 회원가입 인증 / 비밀번호 재설정 / 알림 / 마케팅 메일의 표준 백엔드.

---

## 2. 왜 / 언제

- 응용에서 트랜잭션 이메일 (가입 인증 / 비밀번호 / 영수증)
- 마케팅 메일 (대량 발송, bounce / complaint 관리)
- SMTP relay 서버 직접 운영하기 싫을 때
- 인박스 도달률 (SPF / DKIM / DMARC) 매니지드로
- 발송 비용 비교 — Mailgun / SendGrid 대안 (저렴)

---

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Verified identity** | 발송 가능한 도메인 / 이메일 — DNS 인증 필요 |
| **Sandbox / Production** | 신규 account 는 sandbox (검증된 수신자만) → AWS 에 production access 요청 |
| **Configuration Set** | 발송 설정 묶음 (이벤트 publishing / IP pool / TLS) |
| **Sending quota / Send rate** | 일/시간당 한도 (warming 시 점진 증가) |
| **Bounce / Complaint** | 반송 / 스팸 신고 — SNS 로 받아 처리 (필수) |
| **DKIM / SPF / DMARC** | 인증 — 도달률에 직접 영향 |
| **MAIL FROM domain** | bounce 처리용 별도 sub-domain |
| **Dedicated IP** | 전용 IP (대량 발송 시 평판 자체 관리) |

---

## 4. 발송 방법

| 방식 | 설명 |
| --- | --- |
| **API (SDK)** | `SendEmail` / `SendRawEmail` / `SendBulkEmail` |
| **SMTP** | TLS 587 — 기존 응용 / Postfix relay |
| **응용 통합** | Cognito / Pinpoint / CloudWatch Alarm 자동 사용 |

```python
import boto3
ses = boto3.client("sesv2", region_name="ap-northeast-2")
ses.send_email(
    FromEmailAddress="noreply@example.com",
    Destination={"ToAddresses": ["user@y"]},
    Content={"Simple": {
        "Subject": {"Data": "환영합니다"},
        "Body":    {"Text":  {"Data": "가입 완료"}}
    }},
)
```

---

## 5. 표준 패턴 — bounce / complaint 처리

```
SES → SNS topic
        ↓
     Lambda
        ↓
DynamoDB suppression list (메일 발송 금지)
```

→ bounce / complaint 를 무시하면 평판 떨어져 발송 차단됨. **필수 처리**.

---

## 6. 도달률 / 평판

- **SPF + DKIM + DMARC** — 도메인 DNS 에 모두 설정
- **bounce rate < 5%, complaint < 0.1%** 유지
- **warm-up** — 신규 도메인은 점진 증가 (Dedicated IP)
- **content** — 짧고, 정직한 link, image-only 회피
- **List-Unsubscribe header** — 마케팅 메일 필수
- **CloudWatch metrics** — Send / Bounce / Complaint / Delivery 추적

---

## 7. 비용 / 함정

- **EC2 안에서 호출** — $0 인바운드 / $0.10 per 1000 outbound (대략)
- **sandbox 함정** — production access 요청 안 하면 verified 주소로만 발송. 운영 전에 미리 신청.
- **region 제한** — region 별 도달률 영향. 보통 `us-east-1` / `eu-west-1` / `ap-northeast-1` (한국 도착) 사용.
- **첨부파일** — `SendRawEmail` 만 지원. 크기 10MB 이내.
- **SES vs Pinpoint** — Pinpoint = 캠페인 / 세그먼트 / push / SMS. SES = 발송 자체.

---

## 8. 관련

- [[../messaging|↑ Messaging]]
- [[../sns/sns]] — bounce/complaint 알림
- [[../../security/iam/iam|↗ IAM]]
- [[../../network/route53/route53]] — DKIM / SPF DNS
