---
title: "Amazon Inspector"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T11:14:00+09:00
tags:
  - aws
  - security
  - vulnerability
  - inspector
---

# Amazon Inspector

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Inspector hub |

**[[../security|↑ Security]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

EC2 / Lambda / 컨테이너 이미지 (ECR) 의 **CVE 취약점 + 네트워크 노출 자동 스캔**. v2 부터 SBOM 기반, 항상 켜져 있음.

## 2. 왜 / 언제

- 응용 컨테이너 이미지의 라이브러리 CVE
- EC2 OS / 패키지 패치 우선순위
- Lambda 코드 / layer 의 의존성 취약점
- 네트워크 도달성 (EC2 가 인터넷에 어떤 port 로 열려 있나)
- 컴플라이언스 (PCI / HIPAA) 보고

## 3. 스캔 대상

| 대상 | 스캔 |
| --- | --- |
| **EC2** | OS + 패키지 CVE (SSM Agent 필요) + 네트워크 노출 |
| **ECR 이미지** | Docker 이미지 layer CVE (push 시 자동 + 주기적) |
| **Lambda** | 함수 코드 + 의존성 CVE |

## 4. 표준 통합

```
push 이미지 → ECR
              ↓ Inspector 자동 스캔
              ↓
finding → EventBridge → SNS / Lambda
              ↓
Security Hub (집중)
```

→ CI 파이프라인에서 **이미지 high/critical 취약점 → 배포 차단** 자동화.

## 5. 비용 / 함정

- EC2 인스턴스 / 월 (~$1.25), 이미지 스캔 / 이미지 (~$0.09)
- SBOM 추출 — 라이브러리 메타가 없는 컨테이너 (alpine 일부) 는 누락
- finding suppression 룰 — 무관한 finding 은 룰로 묵음
- **Inspector Classic (v1) 단종** — v2 만 사용

## 6. 관련

- [[../security|↑ Security]]
- [[../../compute/ecr/ecr]]
- [[../../compute/ec2/ec2]]
- [[../security-hub/security-hub]]
- [[../guardduty/guardduty]]
