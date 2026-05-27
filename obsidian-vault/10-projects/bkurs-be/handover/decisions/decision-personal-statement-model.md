---
title: "B7B 결정 — PersonalStatement (자소서) + Feedback + DRAFT/SUBMITTED 상태"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T13:00:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, personal-statement, feedback, b7b]
related:
  - "[[../bkurs-be]]"
  - "[[decision-homework-submission-model]]"
home_hub: bkurs-be
---

# B7B 결정 — PersonalStatement + Feedback

## 결론

1. **`personal_statement` (학생 1:N) + `statement_feedback` (statement 1:N)** 별도 테이블.
2. **status enum (DRAFT/SUBMITTED)** — 학생이 명시 submit. SUBMITTED 후 PATCH 차단.
3. **강사 권한 = 본인 ACTIVE enrollment 의 학생** — 본인 수업 수강 학생만 자소서 / 피드백 조회 가능.
4. **피드백은 강사 (또는 OWNER) 작성** — author_account_id audit.

## 왜 필요한가

- 자소서는 학원의 핵심 학습 콘텐츠 (입시 / 전형 / 대학 진학).
- 학생이 초안 / 수정 / 제출 단계 분리 필요 — DRAFT/SUBMITTED.
- 강사 피드백이 학생 학습의 핵심 — 댓글 형식 1:N.

## 4 대안 비교 — 자소서 모델

### A. 단일 statement + revisions JSON — 기각

- 단점: 수정 이력 array 비대 + 검색 어려움.
- **기각**.

### B. statement + feedback 별도 테이블 — **채택**

- 장점: 정규화. statement N + feedback M 양방향.
- 단점: row 비대 (학원 100 × 학생 30 × 자소서 5 = 15k/학원).

### C. statement 가 enrollment FK

- 단점: drop 후 자소서 소실.
- **기각**.

### D. statement revisions 별도 테이블 (이력 보존)

- 미래 후보 (B8+).

## 4 대안 비교 — DRAFT/SUBMITTED 정책

### a. DRAFT/SUBMITTED 분리 + SUBMITTED 후 PATCH 차단 — **채택**

- 장점: 학생 의도 명확 (제출 = 강사 검토 요청). SUBMITTED 후 학생 자의 수정 차단 → 검토 안정성.
- 단점: 학생이 SUBMITTED 후 실수 발견 시 강사에게 reset 요청 필요.

### b. 자유 PATCH (status 없음)

- 단점: 강사 검토 중 학생이 변경 = 작업 무효화.
- **기각**.

### c. 다중 status (DRAFT/SUBMITTED/REVIEWED/FINALIZED)

- 향후 운영 피드백 후. B7B 는 단순 2 단계.

## 테이블 설계

### `personal_statement`

```sql
CREATE TABLE personal_statement (
  id          BIGINT AUTO_INCREMENT PRIMARY KEY,
  account_id  BIGINT NOT NULL,
  title       VARCHAR(200) NOT NULL,
  content     TEXT NOT NULL,
  status      VARCHAR(32) NOT NULL,     -- DRAFT/SUBMITTED
  submitted_at DATETIME(6) NULL,
  created_at  DATETIME(6) NOT NULL,
  updated_at  DATETIME(6) NOT NULL,
  CONSTRAINT fk_statement_account FOREIGN KEY (account_id) REFERENCES account(id) ON DELETE CASCADE,
  KEY idx_statement_account_status (account_id, status)
);
```

### `statement_feedback`

```sql
CREATE TABLE statement_feedback (
  id                  BIGINT AUTO_INCREMENT PRIMARY KEY,
  statement_id        BIGINT NOT NULL,
  author_account_id   BIGINT NOT NULL,
  content             TEXT NOT NULL,
  created_at          DATETIME(6) NOT NULL,
  CONSTRAINT fk_feedback_statement FOREIGN KEY (statement_id) REFERENCES personal_statement(id) ON DELETE CASCADE,
  CONSTRAINT fk_feedback_author FOREIGN KEY (author_account_id) REFERENCES account(id),
  KEY idx_feedback_statement_created (statement_id, created_at)
);
```

## endpoint 설계

| method | path | role | 검증 |
|---|---|---|---|
| POST | /api/v1/students/me/personal-statements | STUDENT | role check |
| GET | /api/v1/students/me/personal-statements | STUDENT | 본인 row |
| PATCH | /api/v1/students/me/personal-statements/{id} | STUDENT | 본인 + DRAFT 만 |
| POST | /api/v1/students/me/personal-statements/{id}/submit | STUDENT | 본인 + DRAFT → SUBMITTED |
| GET | /api/v1/students/me/personal-statements/{id}/feedback | STUDENT | 본인 statement 의 feedback |
| GET | /api/v1/teachers/me/students/{studentId}/personal-statements | TEACHER | 본인 수강생 |
| GET | /api/v1/teachers/me/personal-statements/{id} | TEACHER | 본인 수강생 statement |
| POST | /api/v1/teachers/me/personal-statements/{id}/feedback | TEACHER | 본인 수강생 statement |

## 회귀 라인

`PersonalStatementApiTest` 10:
1. 학생 작성 → DRAFT
2. 학생 본인 목록
3. 학생 submit → SUBMITTED
4. DRAFT PATCH → 200
5. SUBMITTED PATCH → 409
6. 강사 본인 수강생 자소서 GET
7. 강사 피드백 작성 → 201
8. 학생이 본인 statement 의 feedback GET
9. 다른 학원 강사 자소서 → 403
10. 본인 외 학생 statement GET → 403

## 관련

- [[decision-homework-submission-model]] — B7A 와 같은 패턴 (학생 작성 + 강사 피드백)
- 후속: B7C 예상질문 / 기출
