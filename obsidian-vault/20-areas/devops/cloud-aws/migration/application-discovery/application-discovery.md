---
title: "AWS Application Discovery Service"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:24:00+09:00
tags:
  - aws
  - migration
  - discovery
---

# AWS Application Discovery Service

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Application Discovery hub |

**[[../migration|↑ Migration]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

온프레미스 서버 / 응용의 **인벤토리 자동 수집** — VM / 프로세스 / 네트워크 / 성능. 마이그레이션 사전 조사 단계.

## 2. 두 방식

| | Agentless Connector | Agent |
| --- | --- | --- |
| 형태 | OVA (VMware) | 서버마다 설치 |
| 정보 | VM 메타 / 네트워크 | + 프로세스 / 성능 |
| 적합 | VMware 집중 | 정밀 / 다양한 환경 |

## 3. 함정

- 데이터 수집 후 분석은 Migration Hub / Athena 로
- 작은 환경 (5~10 서버) 은 수동 인벤토리가 더 빠름

## 4. 관련

- [[../migration|↑ Migration]]
- [[../migration-hub/migration-hub]]
- [[../mgn/mgn]]
