---
title: "AWS App Runner"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:12:00+09:00
tags:
  - aws
  - compute
  - paas
  - container
---

# AWS App Runner

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | App Runner hub |

**[[../compute|↑ Compute]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

컨테이너 이미지 또는 Git 저장소를 받아 **자동 빌드 / 배포 / HTTPS / 스케일** — Fargate 위의 PaaS. Cloud Run (GCP) / Heroku 의 AWS 버전.

## 2. 왜 / 언제

- 작은 웹서비스 / API 빠르게 (인프라 X)
- ECS Fargate 보다 더 적은 설정
- GitHub push → 자동 배포 + 도메인 + TLS
- 짧은 idle (스케일 to 0 옵션) — Lambda 보다 긴 응답 OK
- 컨테이너이지만 ECS 만큼 제어 안 필요할 때

## 3. 핵심 / 함정

| 개념 | 의미 |
| --- | --- |
| **Service** | 한 응용 — source / port / scale 설정 |
| **Source** | Git (자동 빌드) / 이미지 (ECR / Docker Hub) |
| **Concurrency** | 인스턴스 당 동시 요청 (스케일 트리거) |
| **VPC Connector** | 사설 자원 접근 (RDS 등) |
| **함정** | ECS Fargate 보다 비쌈. 복잡한 routing / sidecar 필요하면 ECS 권장 |

## 4. 관련

- [[../compute|↑ Compute]]
- [[../ecs/ecs]] — 더 정밀한 컨테이너 운영
- [[../lambda/lambda]] — 더 짧고 이벤트 기반
- [[../../frontend/amplify/amplify]] — 풀스택 호스팅
