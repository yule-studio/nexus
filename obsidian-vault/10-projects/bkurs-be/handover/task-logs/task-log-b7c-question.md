---
title: "B7C — Question (예상질문 + 기출) + Attempt 작업 로그"
kind: task-log
project: bkurs-be
session_id: bkurs-be-b7-content
created_at: 2026-05-20T14:00:00+09:00
agent: engineering-agent/tech-lead
status: in-progress
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-question-attempt-model]]"
  - "[[task-log-refactor-abstract-integration-test]]"
tags: [task-log, question, attempt, b7c]
---

# 작업

GitHub issue [#37](https://github.com/seum-running/bkurs-be/issues/37) — Question + Attempt.

# 결정 요약

| 결정 | 내용 |
|---|---|
| 모델 | question 단일 + type enum (EXPECTED/PAST) |
| year/subject | PAST 만, nullable |
| attempt upsert | (question, account) UNIQUE |
| 채점 | TEACHER 본인 배정, score 0-100 + audit |

# 진행

## 2026-05-20
- C1: entity + V16 + AbstractIntegrationTest 2 repo 추가
- C1.5: base augment commit (refactor)
- C2: service 2 + 예외 3
- C3: DTO 6 + controller 2 + ExceptionHandler
- C4: 회귀 10 + CHANGELOG (cleanup 보강 0 곳)
- PR [#38](https://github.com/seum-running/bkurs-be/pull/38) — **162/162 GREEN**

# REFACTOR 효과 확인

| 시점 | 신규 entity 시 cleanup 보강 위치 |
|---|---|
| B6B (이전) | 10 테스트 |
| B7A | 14 테스트 |
| B7B | 15 테스트 |
| **B7C (REFACTOR 후)** | **0 테스트 + AbstractIntegrationTest 1 곳** |

# Handover

| 구현 | 선택 이유 |
|---|---|
| type enum 단일 테이블 | EXPECTED/PAST 가 90% 동일 — schema/service 중복 회피 |
| year/subject nullable | PAST 만 사용 — application validation (controller / DTO) |
| attempt upsert | 학생 재시도 자연스러움. 이력 보존은 B8+. |
| type 필터 query 메서드 4종 | findByCourseId + findByCourseIdAndType + In 버전 — 학생/강사 양측 + 필터 조합 |
| AbstractIntegrationTest 만 보강 | **REFACTOR 패턴 확립 — 17 곳 → 1 곳** |

# 다음

- B7D 모의면접 (MockInterview) 또는 B7E 생기부 (StudentRecord)
- B7F 토론 / 수행평가 / 독서 / 시뮬레이션

# 참고

- [[../decisions/decision-question-attempt-model]]
- [[task-log-refactor-abstract-integration-test]]
- [[task-log-b7a-homework]] — 유사 패턴
