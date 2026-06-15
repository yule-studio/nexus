---
title: "AWS Cloud Map"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:41:00+09:00
tags:
  - aws
  - network
  - service-discovery
---

# AWS Cloud Map

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Cloud Map hub |

**[[../network|↑ Network]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

서비스 디스커버리 — **응용이 동적으로 다른 서비스의 endpoint 찾기**. DNS 기반 (Route 53 private) + API 기반 둘 다.

## 2. 왜 / 언제

- ECS / EC2 인스턴스가 동적으로 등록 / 해제될 때
- ECS Service Connect / Service Discovery 의 백엔드
- 마이크로서비스 endpoint 등록 / 조회

## 3. 함정

- ECS Service Discovery 가 내부적으로 Cloud Map 사용 — 직접 호출보다는 ECS 옵션으로
- 신규 = **VPC Lattice** (cross-VPC L7) 또는 Service Connect 권장

## 4. 관련

- [[../network|↑ Network]]
- [[../route53/route53]]
- [[../../compute/ecs/ecs]]
