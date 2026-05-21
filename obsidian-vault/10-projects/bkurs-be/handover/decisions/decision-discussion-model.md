---
title: "B7F 결정 — Discussion (토론) topic + post + 평가"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T16:30:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, discussion, b7f]
related:
  - "[[../bkurs-be]]"
  - "[[decision-mock-interview-model]]"
home_hub: bkurs-be
---

# B7F 결정 — Discussion (토론)

## 결론

1. **`discussion_topic` (course_id, type enum DEBATE/SEMINAR/QNA) + `discussion_post` (uk topic_id+account_id upsert)** 2 테이블.
2. **position enum (FOR/AGAINST/NEUTRAL)** — DEBATE 에서만 의미, SEMINAR/QNA 는 NEUTRAL default.
3. **upsert** — 학생이 동일 토픽에 재제출하면 update (Question/Interview 패턴 일관).
4. **peer reply 는 본 PR 에서 제외** — single-level posts.
5. **평가**: topic 작성 강사 OR 같은 수업 배정 강사 누구나 (협업 평가 허용).

## 왜 필요한가

- 학원이 토론 수업 진행 → 학생 발화/입장 수집 후 평가.
- 평가 결과 → B7E 생기부 자료.
- 평가 actor 다른 패턴 (Homework grade vs Interview evaluate) 일관: evaluate.

## 4 대안 — peer reply

### A. parent_post_id self-FK — 기각

- 단점: 트리 깊이 무한 → 평가 단위 모호.
- **기각**: 토론 핵심은 발화 수집, 댓글 트리는 부차적.

### B. 단일 레벨 (post 만) — **채택**

- 장점: 평가 단위 명확. 회귀 단순.
- **채택**: PoC 수준 충분.

### C. 별도 reply 테이블

- **기각**: B7F 범위 초과 — 후속 PR.

## 4 대안 — upsert vs append

### A. uk(topic, account) upsert — **채택**

- 장점: Question/Interview 와 일관. 최신 의견 추적.

### B. append-only history

- **기각**: 평가 시점 의견과 분리되어 혼란.

## endpoint 설계

| method | path | role | 검증 |
|---|---|---|---|
| POST | /api/v1/teachers/me/courses/{courseId}/discussions | TEACHER | 본인 배정 |
| GET | /api/v1/teachers/me/courses/{courseId}/discussions | TEACHER | 본인 배정 |
| PATCH | /api/v1/teachers/me/discussions/{id} | TEACHER | 본인 작성 |
| DELETE | /api/v1/teachers/me/discussions/{id} | TEACHER | 본인 작성 |
| POST | /api/v1/students/me/discussions/{id}/posts | STUDENT | enrolled ACTIVE |
| GET | /api/v1/students/me/discussions | STUDENT | enrolled |
| POST | /api/v1/teachers/me/discussion-posts/{postId}/evaluate | TEACHER | 본인 배정 수업 |

## 회귀 10

1. 강사 DEBATE 토픽 생성 → 201
2. 학생 FOR 입장 post → 200
3. 학생 재제출 → upsert (1 row)
4. STUDENT 토픽 생성 → 4xx
5. 다른 학원 강사 → 403
6. 강사 본인 수업 토픽 목록
7. 학생 본인 수업 토픽 목록
8. 강사 평가 (score 90 + 코멘트)
9. PATCH 본인 작성 토픽
10. DELETE topic → post CASCADE

## 관련

- [[decision-mock-interview-model]] — uk upsert + 평가 패턴 재사용
- 후속: peer reply 별도 PR
