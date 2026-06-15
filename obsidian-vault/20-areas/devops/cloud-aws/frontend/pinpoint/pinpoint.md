---
title: "Amazon Pinpoint"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:25:00+09:00
tags:
  - aws
  - frontend
  - marketing
  - pinpoint
---

# Amazon Pinpoint

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Pinpoint hub |

**[[../frontend|↑ Frontend]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**모바일 푸시 / 이메일 / SMS / Voice 마케팅 캠페인**. 세그먼트 / 저니 / A/B 테스트. SES 는 발송 자체, Pinpoint 는 캠페인 / 분석 위.

## 2. 왜 / 언제

- 사용자 세그먼트 (가입 N일 / 결제 / 비활성 등)
- multi-channel 캠페인 (push + 이메일 + SMS)
- journey — 트리거 → wait → 분기
- 캠페인 분석 (open / click)

## 3. 함정

- AWS 가 Pinpoint 일부를 SES / EUM 로 분할하는 추세 — 일부 기능 곧 deprecated
- 이메일 발송 자체 = SES. Pinpoint 는 layer 위
- 대안: Braze / OneSignal / SendGrid (SaaS, 더 풍부)

## 4. 관련

- [[../frontend|↑ Frontend]]
- [[../../messaging/ses/ses]]
- [[../../messaging/sns/sns]]
