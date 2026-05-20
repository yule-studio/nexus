---
title: "B7I task-log — Simulation (모의지원) case + review"
kind: task-log
project: bkurs-be
status: in_review
created_at: 2026-05-20T17:15:00+09:00
agent: engineering-agent/coding
tags: [task-log, simulation, b7i]
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-simulation-model]]"
home_hub: bkurs-be
---

# B7I task-log — Simulation (모의지원)

## 결과

- 이슈: #49
- PR: #50 (draft, stacked on #48 B7H)
- 브랜치: `feature/issue-49-b7i-simulation`
- 4 commit (S1~S4)
- 10 통합 회귀 PASS

## commit 별 산출물

| commit | 내용 | LOC | 비고 |
|---|---|---|---|
| bd3217f | V22 + enum + 2 entity + 2 repo + base | +260 | uk(case,student) |
| b9f2b96 | Service + 예외 3 + DTO 6 | +280 | resubmit review null reset |
| 11e6834 | endpoint 6+2 + ExceptionHandler | +160 | 8 endpoint |
| 8144e31 | SimulationApiTest 10 + CHANGELOG | +222 | 모든 회귀 PASS |

## 8 endpoint

- TEACHER (6) — POST/GET/PATCH/DELETE case + GET responses + POST review
- STUDENT (2) — GET cases + POST respond

## 3 type (SimulationType)

| enum | 의미 |
|---|---|
| UNIV_APPLICATION | 대입 모의지원 |
| CAREER | 진로 선택 |
| FINANCIAL | 재정 시뮬 |

## 10 회귀 (SimulationApiTest)

1. UNIV_APPLICATION case → 201 ✅
2. CAREER 다type ✅
3. STUDENT 거부 (4xx) ✅
4. 다른 학원 강사 403 ✅
5. 강사 list ✅
6. 학생 enrolled list ✅
7. 학생 응답 → 200 ✅
8. 재응답 → upsert + review null reset ✅
9. 강사 review (feasibility 5) ✅
10. DELETE case → response CASCADE ✅

## 선택 이유 — handover

### A. PE 와 별도 도메인

- **선택 이유**: 정답 없음 / 현실성 평가 — PE 의 점수 채점과 의미 다름.
- **기각된 대안**: PE 에 type 추가 (의미 흐려짐).
- **트레이드오프**: schema 중복 — 후속 multi-step / structured choices 확장 시 분리가 유리.

### B. feasibility 1-5 (PE maxScore 와 다른 스케일)

- **선택 이유**: 시뮬은 "현실성" 평가 — 별점 (ReadingNote recommendedScore 와 일관).
- **기각된 대안**: 0-100 PE 패턴 (점수 의미 약함).

### C. decisions + justification 자유 텍스트

- **선택 이유**: 시나리오 다양성 자유 (대입/진로/재정 어떤 형식이든).
- **기각된 대안**: structured choices JSON (첫 PR schema 복잡).
- **future**: 후속 PR 에서 SimulationOption 별도 테이블 + 학생 선택지 매핑.

### D. 재제출 시 review 모든 필드 null reset

- **선택 이유**: PE 패턴 일관 — 다른 내용 재제출 = 이전 review 무효.
- **trade-off**: 강사 재review 부담 — 그러나 명확.

### E. 협업 review (case 작성자 한정 X)

- **선택 이유**: 본인 배정 수업 강사 누구나 review (PE/Discussion 일관).
- **audit**: reviewed_by_account_id 로 책임 추적.

## 관련

- [[../decisions/decision-simulation-model]]
- [[task-log-b7h-performance-evaluation]] — 직전 stack
- 후속: structured choices / multi-step / B8 PDF export
