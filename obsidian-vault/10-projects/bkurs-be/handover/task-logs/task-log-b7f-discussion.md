---
title: "B7F task-log — Discussion (토론) topic + post + 평가"
kind: task-log
project: bkurs-be
status: in_review
created_at: 2026-05-20T16:35:00+09:00
agent: engineering-agent/coding
tags: [task-log, discussion, b7f]
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-discussion-model]]"
home_hub: bkurs-be
---

# B7F task-log — Discussion (토론)

## 결과

- 이슈: #43
- PR: #44 (draft, stacked on #42 B7E)
- 브랜치: `feature/issue-43-b7f-discussion`
- 4 commit (S1~S4)
- 10 통합 회귀 PASS

## commit 별 산출물

| commit | 내용 | LOC | 비고 |
|---|---|---|---|
| df45cb8 | V19 + 2 enum + 2 entity + 2 repo + base 보강 | +280 | uk(topic,account) |
| a1eabda | DiscussionService + 예외 3 + DTO 6 | +260 | upsert + 협업 평가 |
| 978fc4d | endpoint 6+2 + ExceptionHandler | +160 | 8 endpoint |
| ceead3a | DiscussionApiTest 10 + CHANGELOG | +215 | 모든 회귀 PASS |

## 8 endpoint

- TEACHER (6) — POST/GET/PATCH/DELETE `/api/v1/teachers/me/courses/{cid}/discussions` + `/discussions/{tid}` + GET `/discussions/{tid}/posts` + POST `/discussion-posts/{pid}/evaluate`
- STUDENT (2) — GET `/api/v1/students/me/discussions` + POST `/discussions/{tid}/posts`

## 10 회귀 (DiscussionApiTest)

1. 강사 DEBATE 토픽 → 201 ✅
2. 학생 FOR 입장 → 200 ✅
3. 학생 재제출 → upsert 1 row ✅
4. STUDENT 토픽 생성 → 거부 (4xx) ✅
5. 다른 학원 강사 → 403 ✅
6. 강사 본인 수업 토픽 목록 ✅
7. 학생 본인 enrolled 토픽 목록 ✅
8. 강사 평가 (score 90) ✅
9. PATCH 본인 작성 토픽 ✅
10. DELETE topic → post CASCADE ✅

## 선택 이유 — handover

### A. topic + post 2 테이블 (parent_post_id self-FK 기각)

- **선택 이유**: 토론 핵심은 발화 수집, 댓글 트리는 부차적.
- **기각된 대안**: parent_post_id self-FK (트리 깊이 무한 → 평가 단위 모호).
- **트레이드오프**: 학생끼리 reply 불가 — 후속 PR 에서 reply 테이블 별도 추가.

### B. uk(topic_id, account_id) upsert

- **선택 이유**: Question/Interview 패턴 일관. 학생 1인 1 의견 — 최신 추적.
- **기각된 대안**: append-only history (평가 시점과 분리되어 혼란).

### C. 협업 평가 (작성 강사 한정 X)

- **선택 이유**: 학원 내 같은 수업 배정 강사 누구나 평가 가능 — 학원 운영 유연성.
- **기각된 대안**: topic 작성자 한정 (1인 강사 부재 시 평가 불가).
- **트레이드오프**: 누가 평가했는지 audit 필수 (`evaluated_by_account_id`).

### D. DiscussionPosition NEUTRAL default

- **선택 이유**: SEMINAR/QNA 에서는 입장 의미 없음 — NEUTRAL 로 통일.
- **기각된 대안**: position nullable (DEBATE 강제 어려움).

### E. CourseEnrollmentRepository.findByAccountIdAndStatus 추가

- **선택 이유**: 학생 본인 enrolled 수업 통합 조회 필요 (학원 → 수업 단위).
- **기각된 대안**: courseService 에 메서드 추가 (도메인 응집도 떨어짐).
- **재사용**: 후속 도메인 (수행평가/독서/시뮬) 에서 같은 패턴 활용.

## 관련

- [[../decisions/decision-discussion-model]] — topic+post + 8 endpoint + 회귀 설계
- [[task-log-b7e-student-record]] — 직전 stack
- 후속: B7G (수행평가/시뮬/독서) 또는 B8 (PDF export)
