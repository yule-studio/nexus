---
title: "AWS Application Migration Service (MGN)"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T13:21:00+09:00
tags:
  - aws
  - migration
  - mgn
  - lift-and-shift
---

# AWS Application Migration Service (MGN)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | MGN hub |

**[[../migration|↑ Migration]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

VM / 물리 서버를 **EC2 로 lift-and-shift**. 블록 레벨 replication → 짧은 cutover 다운타임. 옛 CloudEndure Migration 의 후신.

## 2. 왜 / 언제

- 데이터센터 → AWS rehost
- VMware / Hyper-V VM 이전
- 무중단 (수 분 cutover) — 응용 코드 수정 X
- AWS DRS 와 같은 엔진 (DR 용도)

## 3. 함정

- 응용 native 클라우드 이점은 없음 (rehost 만)
- 이전 후 점진 replatform / refactor 권장
- 학습 데이터 / 라이센스 (Windows Server) 별도 처리

## 4. 관련

- [[../migration|↑ Migration]]
- [[../migration-hub/migration-hub]]
- [[../../compute/ec2/ec2]]
