---
title: "B7B — PersonalStatement (자소서) + Feedback 작업 로그"
kind: task-log
project: bkurs-be
session_id: bkurs-be-b7-content
created_at: 2026-05-20T13:00:00+09:00
agent: engineering-agent/tech-lead
status: in-progress
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-personal-statement-model]]"
  - "[[task-log-b7a-homework]]"
tags: [task-log, personal-statement, b7b]
---

# 작업

GitHub issue [#33](https://github.com/seum-running/bkurs-be/issues/33) — 자소서 + 피드백.

# 결정 요약

| 결정 | 내용 |
|---|---|
| 모델 | statement (학생 1:N) + feedback (statement 1:N) 별도 |
| 상태 | DRAFT/SUBMITTED, SUBMITTED 후 PATCH 차단 |
| 강사 권한 | 본인 ACTIVE 수강생만 |
| 피드백 | 강사 1:N 댓글, author audit |

# 진행

## 2026-05-20
- C1: entity + V15 + invariant
- C2: 2 service + 2 예외
- C3: 5 DTO + 2 controller + ExceptionHandler
- C4: 10 회귀 + 15 cleanup 보강 + CHANGELOG
- PR [#34](https://github.com/seum-running/bkurs-be/pull/34) — **152/152 GREEN**

# 인수인계 (Handover)

| 구현 | 선택 이유 |
|---|---|
| DRAFT/SUBMITTED 2 단계 | 학생 의도 명확 (제출 = 검토 요청) + SUBMITTED 안정성 |
| SUBMITTED PATCH 차단 (IllegalState → 409) | 강사 검토 중 학생 변경 = 작업 무효화 방지 |
| 강사 검증 = listByTeacher × enrollment ACTIVE | B6/B7A 와 같은 패턴 — 일관성 |
| StudentStatementController role 무관 | service 가 STUDENT 검증, 학생 외 시도 403 statement_not_yours |
| feedback 단순 1:N | threading / mention 은 B8+ 별도 결정 |
| `loadMine` 도메인 헬퍼 | ownership 검증 응집 — service 의 모든 학생 endpoint 가 위임 |
| `ensureStudentInTeacherCourses` cross-domain | statement → student → enrollment chain. 강사 권한 검증. |
| 15 테스트 cleanup 반복 — **BaseAcademyTest 강력 후보** | B7B 도입으로 15 곳 동일 cleanup. 다음 PR 전 추출 강력 권장. |

# 다음 단계

- B7C: 예상질문 / 기출 (ExpectedQuestion + PastQuestion)
- 또는 **BaseAcademyTest 헬퍼 추출** (refactor PR) — 15 테스트 cleanup 반복 해소
- B7D: 모의면접 / B7E: 생기부 / B7F: 토론·수행평가·독서·시뮬

# 회수 / pitfall

- 15 테스트 cleanup 반복 패턴 — BaseAcademyTest 추출이 B7C 도입 전 우선순위.

# 참고

- [[../decisions/decision-personal-statement-model]]
- [[task-log-b7a-homework]]
