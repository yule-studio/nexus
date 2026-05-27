---
title: "B7C 결정 — Question (예상질문 + 기출 통합 type) + Attempt upsert"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T14:00:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, question, attempt, b7c]
related:
  - "[[../bkurs-be]]"
  - "[[decision-homework-submission-model]]"
home_hub: bkurs-be
---

# B7C 결정 — Question + Attempt

## 결론

1. **`question` 단일 테이블 + `type` enum (EXPECTED/PAST)** — 두 종류 통합. PAST 는 `year`/`subject` 옵셔널.
2. **`question_attempt` 별도 (question 1:N) + `(question, account)` UNIQUE → upsert.**
3. **채점은 강사 (또는 OWNER)** — score 0-100 + feedback + scored_by audit.
4. **권한**: 강사 본인 배정 수업 (CRUD + 채점), 학생 본인 ACTIVE 수강 수업 (조회 + attempt).

## 왜 필요한가

- 예상질문 / 기출 = 학원 학습 콘텐츠 핵심 (입시 준비 / 면접 대비).
- 두 종류 모델이 거의 동일 (출제/풀이/채점) — 단일 테이블 + type 으로 통합 단순.
- 학생이 attempt → 강사 채점 워크플로우 (B7A homework 와 동등).

## 4 대안 비교 — Question 모델

### A. expected_question + past_question 별도 테이블 — 기각

- 단점: schema/service/endpoint 중복. 90% 컬럼 동일.
- **기각**.

### B. question 단일 + type enum — **채택**

- 장점: 단일 schema/service. type 필터로 분기.
- 단점: type 별 컬럼 차이 (year/subject 는 PAST 만) — nullable 처리.

### C. inheritance (single table inheritance JPA)

- 단점: JPA inheritance 복잡도 + dtype 컬럼 자동 — 직접 enum 컬럼이 단순.
- **기각**.

### D. JSON 컬럼 (question_data)

- 단점: 검색/검증 어려움.
- **기각**.

## 4 대안 비교 — attempt 모델

### a. attempt 1 row per attempt (upsert) — **채택**

- 장점: 학생 재시도 = update. 단순.

### b. attempt 다중 row per (question, account)

- 장점: 시도 이력 보존.
- 단점: B7C 범위 초과. 추후 attempt_revision 별도 검토.

## 테이블 설계

### `question`

```sql
CREATE TABLE question (
  id                     BIGINT AUTO_INCREMENT PRIMARY KEY,
  course_id              BIGINT NOT NULL,
  type                   VARCHAR(32) NOT NULL,     -- EXPECTED / PAST
  author_account_id      BIGINT NOT NULL,
  title                  VARCHAR(300) NOT NULL,
  body                   TEXT NOT NULL,
  answer                 TEXT NULL,
  year                   INT NULL,                 -- PAST 만
  subject                VARCHAR(100) NULL,        -- PAST 만 (수능/모의/내신 등)
  created_at             DATETIME(6) NOT NULL,
  updated_at             DATETIME(6) NOT NULL,
  CONSTRAINT fk_question_course FOREIGN KEY (course_id) REFERENCES course(id) ON DELETE CASCADE,
  CONSTRAINT fk_question_author FOREIGN KEY (author_account_id) REFERENCES account(id),
  KEY idx_question_course_type (course_id, type)
);
```

### `question_attempt`

```sql
CREATE TABLE question_attempt (
  id                  BIGINT AUTO_INCREMENT PRIMARY KEY,
  question_id         BIGINT NOT NULL,
  account_id          BIGINT NOT NULL,
  content             TEXT NOT NULL,
  submitted_at        DATETIME(6) NOT NULL,
  score               INT NULL,
  feedback            TEXT NULL,
  scored_at           DATETIME(6) NULL,
  scored_by_account_id BIGINT NULL,
  CONSTRAINT fk_attempt_question FOREIGN KEY (question_id) REFERENCES question(id) ON DELETE CASCADE,
  CONSTRAINT fk_attempt_account FOREIGN KEY (account_id) REFERENCES account(id),
  CONSTRAINT uk_attempt UNIQUE (question_id, account_id),
  KEY idx_attempt_account (account_id),
  KEY idx_attempt_question_scored (question_id, scored_at)
);
```

## endpoint 설계

| method | path | role | 검증 |
|---|---|---|---|
| POST | /api/v1/teachers/me/courses/{cid}/questions | TEACHER | 본인 배정 |
| GET | /api/v1/teachers/me/courses/{cid}/questions?type=EXPECTED|PAST | TEACHER | 본인 배정, type 필터 옵션 |
| PATCH | /api/v1/teachers/me/questions/{id} | TEACHER | 본인 배정 |
| DELETE | /api/v1/teachers/me/questions/{id} | TEACHER | 본인 배정 |
| GET | /api/v1/teachers/me/questions/{id}/attempts | TEACHER | 본인 배정 |
| POST | /api/v1/teachers/me/attempts/{id}/score | TEACHER | 본인 배정 |
| GET | /api/v1/students/me/questions?type=... | STUDENT | 본인 ACTIVE 수강 + type 필터 |
| POST | /api/v1/students/me/questions/{id}/attempt | STUDENT | 본인 수강 |
| GET | /api/v1/students/me/attempts | STUDENT | 본인 이력 |

## 회귀 라인

10:
1. 강사 EXPECTED 출제 → 201
2. 강사 PAST 출제 (year 2024 + subject "수능")
3. STUDENT 출제 → 403
4. 본인 수강 question 목록
5. type=EXPECTED 필터
6. 학생 attempt → upsert
7. 강사 채점 (score 0-100)
8. 다른 학원 강사 → 403
9. DELETE → attempt CASCADE
10. 학생 본인 attempts 이력

## 관련

- [[decision-homework-submission-model]] — 거의 동일 패턴 (출제/풀이/채점)
- 후속: B7D 모의면접 / B7E 생기부
