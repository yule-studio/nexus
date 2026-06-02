---
title: "AWS App Mesh"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:14:00+09:00
tags:
  - aws
  - containers
  - service-mesh
  - envoy
---

# AWS App Mesh

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | App Mesh hub |

**[[../compute|↑ Compute]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

ECS / EKS / EC2 의 **서비스 메시 (Envoy 기반)** — 마이크로서비스 트래픽 라우팅 / 재시도 / 회로 차단 / 관측. Istio / Linkerd 의 AWS 매니지드 대응. **2026 새 워크로드는 ECS Service Connect / VPC Lattice 권장**.

## 2. 왜 / 언제

- 수십+ 마이크로서비스 간 트래픽 관리
- canary / weighted routing 정밀 제어
- mTLS / 재시도 / 회로 차단을 응용 코드 밖에서

## 3. 함정 / 대안

- **점차 deprecated 추세** — AWS 가 VPC Lattice / ECS Service Connect 로 이동
- 새 프로젝트 = **VPC Lattice (cross-VPC L7 routing)** 또는 ECS Service Connect 권장
- App Mesh 자체는 무료, sidecar (Envoy) 의 자원 비용만

## 4. 관련

- [[../compute|↑ Compute]]
- [[../ecs/ecs]]
- [[../eks/eks]]
