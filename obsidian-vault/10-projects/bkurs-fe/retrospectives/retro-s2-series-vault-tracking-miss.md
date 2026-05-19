---
title: "bkurs-fe S2 시리즈 회고 — vault 기록 누락"
kind: retrospective
project: bkurs-fe
session_id: bkurs-fe-s2-series
created_at: 2026-05-20T00:25:00+09:00
agent: engineering-agent/tech-lead
status: open
related:
  - "[[../bkurs-fe]]"
  - "[[../task-logs/task-log-s2-series-summary]]"
  - "[[../../../memory/feedback_vault_project_tracking]]"
  - "[[../../../memory/feedback_frontend_areas_usage]]"
tags: [retrospective, vault, process-miss]
---

# 무엇이 일어났나

bkurs-fe S2A ~ S2N (28 PR) 동안 vault 기록을 **한 번도 안 함**.

- 10-projects/bkurs-fe/ 폴더 자체가 없었음
- 20-areas/frontend/ 의 react / tailwind / jotai / tanstack-query 노트 보강 0회
- memory 의 [[../../../memory/feedback_frontend_areas_usage|FE areas 활용 정책]] 이 있었지만 invoke 안 함

# 왜 일어났나

1. **PR 단위 작업 흐름이 너무 명확** — issue 생성 → commit × N → PR → CI → 머지 → 다음 PR. vault 가 끼어들 자연스러운 지점이 없었음
2. **memory 의 정책이 약함** — "FE 작업 전 vault 확인" 만 있고 "PR 머지 전 vault 업데이트" 의 강제력 약함
3. **사용자가 도중 지적 안 함** — 28 PR 동안 한 번도 "vault 안 적네?" 없었음 → 작업 흐름 잡힌 후 정정 안 됨
4. **시간 압박** — "다음 작업 진행해줘" 패턴이 28번 반복되며 작업 끝내는 데만 집중

# 결과

- 28 PR 의 의사결정 / 패턴 / pitfall 이 **PR description 안에만** 남음 (vault 검색 불가)
- 새 작업자 (또는 본인이 6개월 후) 가 같은 결정을 다시 해야 — 무엇을 왜 했는지 가 PR 본문 28개 흩어짐

# 무엇을 바꿀까

## 즉시 (2026-05-20)

- [[../../../memory/feedback_vault_project_tracking|vault project tracking 정책 memory]] 신설 (사용자 요청)
- 사후라도 [[../bkurs-fe|hub]] / [[../task-logs/task-log-s2-series-summary|시리즈 task-log]] / [[../decisions/decision-vite-react-19-tailwind-antd-jotai|stack 결정]] / 본 회고 작성
- bkurs-be 는 PR 단위로 vault 동반 업데이트 (시작부터 적용)

## 정책 강화

- PR 머지 전 체크리스트에 "vault 노트 1개 이상 추가/보강" 추가
- 새 의존성 / 새 패턴 도입 시 20-areas 의 관련 노트 보강 의무화
- 같은 패턴 3회 이상 재사용 → 40-patterns 로 추출

# 회수 작업 (TODO, 우선순위 낮음)

- 28 PR description 을 훑어 핵심 결정 / pitfall 만 vault 의 적절한 위치로 옮기기
- AiAnalysisPanel / ThreadView / ScoreRadar 의 재사용 패턴 → 40-patterns/frontend/react-domain-reuse-patterns.md 추출

# 참고

- [[../bkurs-fe|프로젝트 hub]]
- [[../task-logs/task-log-s2-series-summary]]
- [[../../../memory/feedback_vault_project_tracking]]
