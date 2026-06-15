---
title: "AWS Config"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:20:00+09:00
tags:
  - aws
  - management
  - compliance
  - config
---

# AWS Config

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Config hub |

**[[../management|↑ Management]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

AWS 자원의 **설정 변경 이력 + 규정 준수 룰 자동 평가**. "S3 버킷이 public 이 되면 알람", "EBS 가 암호화 안 되면 자동 수정" 같은 거버넌스.

## 2. 왜 / 언제

- 컴플라이언스 — CIS / PCI / HIPAA 룰 자동 점검
- 변경 추적 — 누가 / 언제 / 어떻게 SG 룰을 바꿨나
- drift 감지 — IaC 와 실제 상태 차이
- 자동 remediation — 위반 자원 자동 수정 (SSM Automation)
- 멀티 account 거버넌스 — Organizations 위에서 통합

## 3. 핵심 개념

| 개념 | 의미 |
| --- | --- |
| **Configuration Recorder** | 자원 상태 / 변경 기록 (계속) |
| **Config Rule** | 평가 룰 (AWS managed 200+ 또는 custom Lambda) |
| **Conformance Pack** | 룰 묶음 (예: "PCI DSS pack") |
| **Aggregator** | 멀티 account / region finding 통합 |
| **Remediation** | 위반 시 SSM Automation 자동 실행 |
| **Resource timeline** | 자원의 모든 변경 이력 시각화 |

## 4. 표준 룰 예시 (AWS Managed)

- `s3-bucket-public-read-prohibited`
- `encrypted-volumes`
- `iam-password-policy`
- `restricted-ssh` (SG 의 22 port 0.0.0.0/0 차단)
- `rds-storage-encrypted`
- `vpc-flow-logs-enabled`

## 5. 표준 통합

```
변경 → Config Recorder → Rule 평가
                          ↓
                    NON_COMPLIANT
                          ↓
                  EventBridge → SNS / Slack
                          ↓ (선택)
                  SSM Automation → 자동 수정
```

## 6. 비용 / 함정

- 자원 변경 1건 $0.003, 룰 평가 $0.001 — **활성 자원 많으면 빠르게 누적**
- 모든 자원 추적 (`Record all`) vs 선택 추적 — 큰 차이
- region 별 활성화 필요 — Organizations 의 Aggregator 추천
- finding 만 만들고 액션 없으면 무용 — remediation 또는 알림 필수

## 7. 관련

- [[../management|↑ Management]]
- [[../control-tower/control-tower]]
- [[../../security/security-hub/security-hub]]
- [[../../observability/cloudtrail/cloudtrail]]
