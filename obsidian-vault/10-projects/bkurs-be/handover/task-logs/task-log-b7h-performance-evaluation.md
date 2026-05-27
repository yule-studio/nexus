---
title: "B7H task-log — PerformanceEvaluation (수행평가) + 채점"
kind: task-log
project: bkurs-be
status: in_review
created_at: 2026-05-20T16:55:00+09:00
agent: engineering-agent/coding
tags: [task-log, performance-evaluation, b7h]
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-performance-evaluation-model]]"
home_hub: bkurs-be
---

# B7H task-log — PerformanceEvaluation (수행평가)

## 결과

- 이슈: #47
- PR: #48 (draft, stacked on #46 B7G)
- 브랜치: `feature/issue-47-b7h-performance-evaluation`
- 4 commit (S1~S4)
- 10 통합 회귀 PASS

## commit 별 산출물

| commit | 내용 | LOC | 비고 |
|---|---|---|---|
| 80a0f47 | V21 + enum + 2 entity + 2 repo + base 보강 | +260 | resubmit 채점 무효화 |
| 6e9cc4f | Service + 예외 3 + DTO 6 | +280 | score 0-maxScore 검증 |
| 09278c6 | endpoint 6+2 + ExceptionHandler | +160 | 8 endpoint |
| 344da49 | PerformanceEvaluationApiTest 10 + CHANGELOG | +221 | 모든 회귀 PASS |

## 8 endpoint

- TEACHER (6) — POST/GET/PATCH/DELETE eval + GET submissions + POST grade
- STUDENT (2) — GET evals + POST submit

## 4 카테고리 (PerformanceCategory)

| enum | 의미 |
|---|---|
| WRITTEN | 서술형 |
| PROJECT | 프로젝트 |
| PRESENTATION | 발표 |
| EXPERIMENT | 실험 |

## 10 회귀 (PerformanceEvaluationApiTest)

1. 강사 WRITTEN 출제 → 201 ✅
2. 강사 PROJECT (maxScore 50) ✅
3. STUDENT 출제 → 거부 (4xx) ✅
4. 다른 학원 강사 → 403 ✅
5. 강사 본인 수업 list ✅
6. 학생 enrolled list ✅
7. 학생 제출 → 200 ✅
8. 재제출 → upsert + 채점 무효화 ✅
9. 강사 채점 (score 95) ✅
10. DELETE eval → submission CASCADE ✅

## 선택 이유 — handover

### A. Homework 와 별도 도메인

- **선택 이유**: 수행평가 = 공식 (생기부 SUBJECT 자료), Homework = 일상. 의미 분리.
- **기각된 대안**: Homework 에 category enum 추가 (의미 흐려짐).
- **트레이드오프**: schema 중복 — 후속 rubric/auto-record 확장 시 분리가 유리.

### B. 재제출 시 채점 무효화

- **선택 이유**: 채점 후 재제출 = 다른 내용, 이전 점수 무효 — 재채점 강제.
- **기각된 대안**: 점수 유지 (학생 재제출 후 헌 점수로 학생부 → 오류 위험).
- **trade-off**: 강사가 재채점 부담 — 그러나 명확.

### C. maxScore 1-1000 invariant + score 0-maxScore 강제

- **선택 이유**: entity 단위 검증 — DTO 우회 (entity 직접 update) 시도 차단.
- **기각된 대안**: DTO 만 (entity 직접 사용 시 검증 누락).

### D. 협업 채점 (작성자 한정 X)

- **선택 이유**: 본인 배정 수업 강사 누구나 채점 (Discussion 일관).
- **기각된 대안**: eval 작성자 한정 — 1인 강사 부재 시 채점 불가.
- **audit**: graded_by_account_id 로 책임 추적.

### E. PerformanceCategory 4종 (확장 가능)

- **선택 이유**: 학종 자료 작성 시 유형별 분류 (WRITTEN/PROJECT/PRESENTATION/EXPERIMENT).
- **기각된 대안**: free-text type (검색/통계 어려움).
- **future**: TOPIC_DEBATE / GROUP_TASK 등 추가는 enum 확장.

## 관련

- [[../decisions/decision-performance-evaluation-model]]
- [[task-log-b7g-reading-note]] — 직전 stack
- 후속: rubric (criteria_json) / 자동 StudentRecordEntry SUBJECT 생성 / B8 PDF export
