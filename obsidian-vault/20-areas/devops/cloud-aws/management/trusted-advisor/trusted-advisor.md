---
title: "AWS Trusted Advisor"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:22:00+09:00
tags:
  - aws
  - management
  - best-practice
  - trusted-advisor
---

# AWS Trusted Advisor

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Trusted Advisor hub |

**[[../management|↑ Management]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

AWS 가 자동으로 점검하는 **베스트 프랙티스 체크리스트** — 비용 / 성능 / 보안 / 한도 / 운영 우수성 5 카테고리, 룰 ~100 개. Business+ 지원 플랜 필수.

## 2. 왜 / 언제

- 정기 운영 헬스체크
- 미사용 EIP / EBS / Idle LB 비용 절감
- 노출된 SG / IAM access key 발견
- 서비스 한도 (quota) 임박 알림
- Business / Enterprise 지원 플랜의 핵심 도구

## 3. 5 카테고리

| 카테고리 | 예시 룰 |
| --- | --- |
| **Cost Optimization** | 미사용 EIP / Idle LB / underutilized EC2 / RDS |
| **Performance** | 큰 EBS / over-provisioned EC2 / overutilized EBS |
| **Security** | public SG / root MFA 없음 / public S3 / 노출된 access key |
| **Fault Tolerance** | EBS 스냅샷 / RDS Multi-AZ / Auto Scaling |
| **Service Limits** | EC2 인스턴스 / EBS / RDS / VPC 한도 80% 이상 |

## 4. 비용 / 함정

- **Basic / Developer 지원** — Core checks 만 (7개)
- **Business / Enterprise** — **모든 체크 + API + EventBridge 알림**
- 단순 알림만 — 자체 차단은 안 함. EventBridge + Lambda 로 자동화
- 일부 룰은 24h cache — 변경 후 바로 안 보임
- Organizations 통합 — management 가 member 의 finding 도 통합 view

## 5. 관련

- [[../management|↑ Management]]
- [[../config/config]] — 더 정밀한 룰 정의
- [[../../cost/cost]] — 비용 최적화 끝에는 Cost Explorer + Compute Optimizer
