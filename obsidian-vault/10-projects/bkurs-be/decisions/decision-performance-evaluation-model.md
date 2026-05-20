---
title: "B7H 결정 — PerformanceEvaluation (수행평가) 카테고리 + 채점"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T16:55:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, performance-evaluation, b7h]
related:
  - "[[../bkurs-be]]"
  - "[[decision-homework-model]]"
home_hub: bkurs-be
---

# B7H 결정 — PerformanceEvaluation (수행평가)

## 결론

1. **`performance_evaluation` + `performance_submission` 2 테이블** — Homework 와 schema 유사하지만 도메인 분리.
2. **category enum 4종** (WRITTEN/PROJECT/PRESENTATION/EXPERIMENT) — Homework 에 없는 분류.
3. **uk(eval_id, student_id) upsert** — 학생 재제출 = update.
4. **max_score nullable default 100** — entity 별 채점 상한.
5. **graded_at / submitted_at audit** — 마감 추적.

## 왜 필요한가

- Homework 는 일상 학습 (점수 가벼움), 수행평가는 공식 (생기부 SUBJECT 연계).
- 카테고리별 분류 → 학종 자료 작성 시 유형 추적.
- 점수 / 피드백 정식 audit (graded_by).

## 4 대안 — Homework 와의 분리

### A. Homework 에 category 컬럼 추가 — 기각

- 단점: 의미 다른 도메인 한 테이블 — service / endpoint 분기 복잡.
- **기각**.

### B. 별도 도메인 (PerformanceEvaluation) — **채택**

- 장점: 도메인 의미 명확, 후속 rubric / 자동 record 연계 자유로움.
- **채택**.

### C. 단일 "Assignment" 통합 + type enum

- **기각**: Homework/PE 두 개념을 한 테이블에 — 의도 약화.

## 4 대안 — rubric (criteria 세부 점수)

### A. criteria_json TEXT 컬럼

- 장점: 한 테이블, 빠른 read.
- 단점: SQL 검색 약함.

### B. 별도 criterion 테이블

- 장점: 정규화.
- 단점: 4 테이블 (eval/criterion/submission/score) — 첫 PR 과도.

### C. 단일 max_score / score — **채택 (1차)**

- 장점: PoC 단순, 후속 PR 로 rubric 확장 자유.
- **채택**: 회귀 단순화.

## endpoint 설계

| method | path | role | 검증 |
|---|---|---|---|
| POST | /api/v1/teachers/me/courses/{cid}/performance-evaluations | TEACHER | 본인 배정 |
| GET | /api/v1/teachers/me/courses/{cid}/performance-evaluations | TEACHER | 본인 배정 |
| PATCH | /api/v1/teachers/me/performance-evaluations/{eid} | TEACHER | 본인 작성 |
| DELETE | /api/v1/teachers/me/performance-evaluations/{eid} | TEACHER | 본인 작성 |
| GET | /api/v1/teachers/me/performance-evaluations/{eid}/submissions | TEACHER | 본인 배정 |
| POST | /api/v1/teachers/me/performance-submissions/{sid}/grade | TEACHER | 본인 배정 수업 |
| GET | /api/v1/students/me/performance-evaluations | STUDENT | enrolled 통합 |
| POST | /api/v1/students/me/performance-evaluations/{eid}/submit | STUDENT | enrolled |

## 회귀 10

1. 강사 WRITTEN 출제 → 201
2. 강사 PROJECT 출제 (다른 카테고리)
3. STUDENT 출제 → 거부 (4xx)
4. 다른 학원 강사 → 403
5. 강사 본인 수업 list
6. 학생 본인 enrolled list
7. 학생 제출 → 200
8. 학생 재제출 → upsert
9. 강사 채점 (score 95 + feedback)
10. DELETE eval → submission CASCADE

## 관련

- [[decision-discussion-model]] — uk upsert + 채점 패턴 재사용
- 후속: rubric (criteria_json) / 자동 StudentRecordEntry 생성
