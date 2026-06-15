---
title: "AWS Outposts"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:00:00+09:00
tags:
  - aws
  - compute
  - hybrid
  - outposts
---

# AWS Outposts

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Outposts hub |

**[[../compute|↑ Compute]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**온프레미스 환경에 AWS 인프라를 그대로 설치** — EC2 / EBS / RDS / EKS 등을 자체 데이터센터에서. AWS 가 직접 하드웨어 / 운영.

## 2. 왜 / 언제

- 데이터 주권 / 컴플라이언스로 데이터가 region 밖으로 못 나갈 때
- 저지연 (< 5ms) 이 필요한 응용 (제조 / 의료 / 금융)
- 하이브리드 — 동일 AWS API + IaC 사용
- 매우 비싸고 큰 약정 (1년/3년) — Enterprise 만

## 3. 함정

- 랙 단위 약정 — 큰 자본 투자
- 일부 서비스만 지원 (EC2 / EBS / RDS / EKS / ECS 등 한정)
- 작은 워크로드 = Local Zones / Wavelength 또는 직접 클라우드

## 4. 관련

- [[../compute|↑ Compute]]
- [[../local-zones/local-zones]]
- [[../wavelength/wavelength]]
