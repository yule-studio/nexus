---
title: "B7G task-log — ReadingNote (독서 기록) + Feedback"
kind: task-log
project: bkurs-be
status: in_review
created_at: 2026-05-20T16:45:00+09:00
agent: engineering-agent/coding
tags: [task-log, reading-note, b7g]
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-reading-note-model]]"
home_hub: bkurs-be
---

# B7G task-log — ReadingNote (독서 기록)

## 결과

- 이슈: #45
- PR: #46 (draft, stacked on #44 B7F)
- 브랜치: `feature/issue-45-b7g-reading-note`
- 4 commit (S1~S4)
- 10 통합 회귀 PASS

## commit 별 산출물

| commit | 내용 | LOC | 비고 |
|---|---|---|---|
| a673f7e | V20 + 2 entity + 2 repo + base 보강 | +220 | uk(note,teacher) |
| ba16833 | ReadingService + 예외 2 + DTO 5 | +220 | student/teacher 분리 listFeedback |
| 6821576 | endpoint 5+2 + ExceptionHandler | +130 | 7 endpoint |
| 62257b7 | ReadingNoteApiTest 10 + CHANGELOG | +227 | 모든 회귀 PASS |

## 7 endpoint

- STUDENT (5) — POST/GET/PATCH/DELETE `/api/v1/students/me/reading-notes` + GET `/.../{nid}/feedback`
- TEACHER (2) — GET `/api/v1/teachers/me/students/{sid}/reading-notes` + POST `/.../reading-notes/{nid}/feedback`

## 10 회귀 (ReadingNoteApiTest)

1. 학생 노트 작성 → 201 ✅
2. 학생 본인 목록 ✅
3. 학생 PATCH 본인 작성 ✅
4. 다른 학생 노트 PATCH → 403 ✅
5. DELETE → feedback CASCADE ✅
6. TEACHER 본인 수강생 노트 조회 ✅
7. 다른 수업 학생 조회 → 403 ✅
8. 강사 feedback (score 4) ✅
9. 같은 강사 재 feedback → upsert ✅
10. 다른 강사 같은 노트 feedback (별도 row) ✅

## 선택 이유 — handover

### A. 학생 owner (academy 격리 없음)

- **선택 이유**: 독서는 학생 평생 누적 — 학원 옮겨도 본인 자료 유지.
- **기각된 대안**: academy_id 추가 (학원 이동 시 데이터 파편화).
- **트레이드오프**: 학원 단위 통계 불가 — 강사가 학생별로 조회만.

### B. ReadingFeedback uk(note, teacher) upsert

- **선택 이유**: 강사가 의견 갱신 — Discussion/Question 일관 패턴.
- **기각된 대안**: append-only history (여러 코멘트 UI 복잡).
- **다른 강사 별도 row**: 동시에 여러 강사 평가 가능 — DEBATE 협업 평가와 다른 점.

### C. recommended_score 0-5

- **선택 이유**: 미디어 별점 수준 — 책 추천도 직관적.
- **기각된 대안**: 0-100 (시험 점수 느낌 — 추천 표현에 과함).
- **nullable**: 점수 없이 코멘트만도 가능.

### D. listFeedback 학생/강사 분리

- **선택 이유**: 학생은 본인 노트만 (isOwnedBy), 강사는 enrolled 학생 만 (ownership).
- **기각된 대안**: 통합 + role 분기 (logic 섞임).
- **trade-off**: service method 2개 — 명시적 ownership 검증.

### E. pages 검증 (>=0)

- **선택 이유**: 음수 페이지 의미 없음 — entity 단위 invariant.
- **기각된 대안**: @PositiveOrZero DTO 만 (entity 직접 update 시 검증 누락 가능).

## 관련

- [[../decisions/decision-reading-note-model]] — 학생 owner + uk feedback 설계
- [[task-log-b7f-discussion]] — 직전 stack
- 후속: B7H (수행평가) / B7I (시뮬) / B8 (PDF export)
