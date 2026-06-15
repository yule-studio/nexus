---
title: "AWS Local Zones"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T14:01:00+09:00
tags:
  - aws
  - compute
  - edge
  - local-zones
---

# AWS Local Zones

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | Local Zones hub |

**[[../compute|↑ Compute]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

대도시 가까이 둔 **mini region** — 로스앤젤레스 / 시카고 / 도쿄 등 사용자 가까이 EC2 / EBS / RDS / ALB 일부. 저지연 (< 10ms).

## 2. 왜 / 언제

- 실시간 게임 / 미디어 / 광고 입찰 (저지연 필요)
- 5G / VR / 라이브 스트리밍
- 응용을 사용자 가까이 — region 보다 가까운 곳

## 3. 함정

- 지원 region / 도시 한정 (확인 필요)
- 일부 AWS 서비스만 (EC2 / EBS / RDS / ALB / Application Auto Scaling 정도)
- 큰 응용은 region + Local Zones 혼합

## 4. 관련

- [[../compute|↑ Compute]]
- [[../outposts/outposts]]
- [[../wavelength/wavelength]]
- [[../../network/global-accelerator/global-accelerator]]
