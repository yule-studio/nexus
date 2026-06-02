---
title: "Amazon GuardDuty"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:13:00+09:00
tags:
  - aws
  - security
  - threat-detection
  - guardduty
---

# Amazon GuardDuty

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | GuardDuty hub |

**[[../security|↑ Security]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

AWS 계정의 **이상 행동 / 위협 자동 탐지**. CloudTrail / VPC Flow Logs / DNS Logs / EKS audit / S3 / Lambda / RDS / EBS 를 ML 로 분석해 finding 생성.

## 2. 왜 / 언제

- 침해된 IAM credential 의 비정상 호출 탐지 (지리적 / 시간적 이상)
- crypto-mining 멀웨어 (EC2 통신 패턴)
- S3 버킷에서의 정찰 / 데이터 유출
- EKS pod 의 알려진 악성 IP 통신
- 컨테이너 / Lambda 의 멀웨어 스캔

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Finding** | 위협 탐지 결과 (severity Low / Medium / High) |
| **Data Source** | CloudTrail / VPC Flow / DNS / S3 / EKS / Lambda / Malware / RDS |
| **Protection plan** | 추가 모듈 (S3 Protection / EKS Protection / Malware Protection 등) — 일부 유료 |
| **Trusted IP / Threat IP list** | 화이트/블랙 리스트 |
| **Organizations 통합** | management → 모든 member 에 자동 활성화 |

## 4. 표준 통합

```
GuardDuty findings
   ↓ EventBridge
   ↓
┌──────────────────────────────────────┐
Security Hub (집중 view)
SNS (이메일 / Slack 알림)
Lambda (자동 차단: IAM 권한 회수 / SG 격리)
SIEM / Splunk / Datadog (외부)
```

## 5. 비용 / 함정

- **30일 무료 trial**, 이후 CloudTrail 이벤트 / Flow log 데이터 기반 과금
- 작은 account 도 월 $10~50, 큰 account 는 $100~1000+
- **모든 region 활성화** — 사용 안 하는 region 도 공격 대상
- **Malware Protection / EKS / Lambda Protection** 별도 모듈 — 필요만큼만
- finding 자체는 차단 X — Lambda / EventBridge 로 액션 자동화

## 6. 관련

- [[../security|↑ Security]]
- [[../../observability/cloudtrail/cloudtrail]]
- [[../security-hub/security-hub]]
- [[../inspector/inspector]]
