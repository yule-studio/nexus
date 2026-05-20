---
title: "B7G 결정 — ReadingNote (독서 기록) + Feedback"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T16:45:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, reading-note, b7g]
related:
  - "[[../bkurs-be]]"
  - "[[decision-student-record-categories]]"
home_hub: bkurs-be
---

# B7G 결정 — ReadingNote (독서 기록) + Feedback

## 결론

1. **`reading_note` (학생 owner) + `reading_feedback` (강사 댓글 upsert)** 2 테이블.
2. **학생 주도** — Homework/Discussion 과 다르게 강사 입력 없이도 작성 가능.
3. **feedback uk(note, teacher)** — 같은 강사 재제출 = update.
4. **recommended_score 0-5** — 미디어 별점 수준 (책 추천도).
5. **academy 격리 없음** — 학생 본인 노트는 학원 무관 누적, 강사 조회는 본인 수업 enrolled 학생만.

## 왜 필요한가

- 생기부 (B7E) READING 카테고리의 원천 자료 — 학생이 책 단위로 자세히 누적.
- 강사가 책 추천 + 평가 → 학생 학습 동기 부여.
- 학원 옮겨도 학생 본인 노트는 유지 (academy_id 없음).

## 4 대안 — owner

### A. 학생 owner — **채택**

- 장점: 학생이 평생 누적. 강사는 코멘트 만.
- **채택**: 본질에 맞음.

### B. 강사 owner

- **기각**: 강사가 학생 책을 대신 입력 — 부자연.

## 4 대안 — feedback

### A. uk(note, teacher) upsert — **채택**

- 장점: 강사가 의견 갱신. Question/Discussion 일관.

### B. append-only

- **기각**: 여러 코멘트 누적 → UI 복잡.

## endpoint 설계

| method | path | role | 검증 |
|---|---|---|---|
| POST | /api/v1/students/me/reading-notes | STUDENT | 본인 |
| GET | /api/v1/students/me/reading-notes | STUDENT | 본인 |
| PATCH | /api/v1/students/me/reading-notes/{id} | STUDENT | 본인 작성 |
| DELETE | /api/v1/students/me/reading-notes/{id} | STUDENT | 본인 작성 |
| GET | /api/v1/teachers/me/students/{sid}/reading-notes | TEACHER | enrolled |
| POST | /api/v1/teachers/me/reading-notes/{nid}/feedback | TEACHER | enrolled |

## 회귀 10

1. 학생 노트 작성 → 201
2. 학생 본인 목록
3. 학생 PATCH 본인 작성
4. 다른 학생 노트 PATCH → 403
5. DELETE → feedback CASCADE
6. TEACHER 본인 수강생 노트 조회
7. 다른 수업 학생 조회 → 403
8. 강사 feedback 작성 (score 4)
9. 같은 강사 재 feedback → upsert
10. 다른 강사 같은 노트 feedback (별도 row)

## 관련

- [[decision-student-record-categories]] — READING 카테고리 원천 자료
- [[decision-discussion-model]] — upsert + 학원 격리 패턴
