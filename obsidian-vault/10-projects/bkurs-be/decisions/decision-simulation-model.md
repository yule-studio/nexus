---
title: "B7I 결정 — Simulation (모의지원) case + response + review"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T17:05:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, simulation, b7i]
related:
  - "[[../bkurs-be]]"
  - "[[decision-performance-evaluation-model]]"
  - "[[decision-student-record-categories]]"
home_hub: bkurs-be
---

# B7I 결정 — Simulation (모의지원 시뮬레이션)

## 결론

1. **`simulation_case` + `simulation_response` 2 테이블** — PE 와 유사하지만 의미 분리.
2. **type enum 3종** (UNIV_APPLICATION/CAREER/FINANCIAL) — 대입 모의지원 / 진로 선택 / 재정 시뮬.
3. **uk(case_id, student_id) upsert** + 재제출 시 review 무효화 (PE 패턴 재사용).
4. **feasibility_score 1-5** — PE 의 maxScore 와 다름. "현실성" 5단계 평가 (별점).
5. **decisions + justification** 자유 텍스트 — 후속 PR 에서 structured choice 가능.

## 왜 필요한가

- 대입 / 진로 / 재정 선택은 학생이 시나리오에서 의사결정 연습 필요.
- 강사 review 가 학생부 CAREER 카테고리 자료 (B7E 연계).
- PE / Homework 와 다른 점: 정답이 없음. "현실성" 평가가 핵심.

## 4 대안 — 의사결정 모델

### A. 자유 텍스트 (decisions + justification) — **채택**

- 장점: 시나리오 다양성 자유. 후속 structured 확장 가능.
- **채택**.

### B. structured choice (JSON array of {option, reason})

- **기각**: 첫 PR 부터 schema 복잡. 후속.

### C. multi-step 의사결정 (시나리오 단계마다 분기)

- **기각**: 시나리오 트리 — entity 4+ 필요. PoC 과도.

## 4 대안 — review score

### A. feasibility 1-5 (별점) — **채택**

- 장점: "현실성" 직관적. ReadingNote recommendedScore 와 일관.

### B. 0-100 (PE 패턴)

- **기각**: 시뮬은 정답 없음 — 점수보다 별점 의미.

## endpoint 설계

| method | path | role | 검증 |
|---|---|---|---|
| POST | /api/v1/teachers/me/courses/{cid}/simulation-cases | TEACHER | 본인 배정 |
| GET | /api/v1/teachers/me/courses/{cid}/simulation-cases | TEACHER | 본인 배정 |
| PATCH | /api/v1/teachers/me/simulation-cases/{id} | TEACHER | 본인 작성 |
| DELETE | /api/v1/teachers/me/simulation-cases/{id} | TEACHER | 본인 작성 |
| GET | /api/v1/teachers/me/simulation-cases/{id}/responses | TEACHER | 본인 배정 |
| POST | /api/v1/teachers/me/simulation-responses/{rid}/review | TEACHER | 본인 배정 수업 |
| GET | /api/v1/students/me/simulation-cases | STUDENT | enrolled |
| POST | /api/v1/students/me/simulation-cases/{cid}/respond | STUDENT | enrolled |

## 회귀 10

1. 강사 UNIV_APPLICATION case → 201
2. 강사 CAREER case (다른 type)
3. STUDENT case 생성 → 거부 (4xx)
4. 다른 학원 강사 → 403
5. 강사 본인 수업 list
6. 학생 enrolled list
7. 학생 응답 → 200
8. 학생 재응답 → upsert + review 무효화
9. 강사 review (feasibility 5)
10. DELETE case → response CASCADE

## 관련

- [[decision-performance-evaluation-model]] — upsert + review 패턴 재사용
- [[decision-student-record-categories]] — CAREER 카테고리 후속 연계
- 후속: structured choices / multi-step / B8 PDF
