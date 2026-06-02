---
title: "AWS Health Dashboard"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:23:00+09:00
tags:
  - aws
  - management
  - operations
  - health
---

# AWS Health Dashboard

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Health hub |

**[[../management|↑ Management]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

AWS 서비스 **장애 / 점검 / 보안 알림** 통합 대시보드. "내 자원 / 내 region" 한정으로 필터된 이벤트 + **Health API + EventBridge** 로 자동화.

## 2. 왜 / 언제

- AWS 서비스 장애 시 영향받는 내 자원 알기
- 예정된 점검 (RDS / EC2 retirement) 사전 대응
- 보안 이벤트 (root key 노출 등) AWS 가 직접 알리는 채널
- Operational Issues — 어느 region / 어느 서비스가 영향받는지 가장 빠른 ground truth

## 3. 두 종류

| | Service Health Dashboard | Personal Health Dashboard (PHD) |
| --- | --- | --- |
| 대상 | **AWS 전체 사용자 공통** | **내 account 한정** |
| URL | status.aws.amazon.com | console |
| 영향 자원 | 일반 표 | 내 자원 ID 별 표시 |
| API | RSS 만 | **AWS Health API** (Business+) |

## 4. 표준 통합

```
PHD 이벤트
  ↓ EventBridge (default bus, source = aws.health)
  ↓
Slack / 이메일 / Lambda / 자동 대응
```

자주 잡을 이벤트:
- `EC2 instance retirement` — 인스턴스 stop/start 예정
- `RDS certificate rotation` — RDS 인증서 만료
- `IAM credentials exposed` — access key 가 GitHub 등에 유출
- `AWS service event` — region / 서비스 장애

## 5. 비용 / 함정

- 무료, AWS Health API 는 **Business / Enterprise 지원** 필요
- Slack 알림 없이 콘솔만 보면 사고 시 늦음 — EventBridge 통합 필수
- Organizations 의 management 가 모든 member 의 PHD 이벤트 통합 view (Org View)
- 어느 region 의 어느 서비스 장애인지 — Health Dashboard 가 가장 빠른 ground truth

## 6. 관련

- [[../management|↑ Management]]
- [[../../observability/observability]]
- [[../trusted-advisor/trusted-advisor]]
