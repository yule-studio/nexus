---
title: "B7A 결정 — Homework + Submission 모델 + 제출 upsert + 강사 채점"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T12:30:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, homework, submission, b7a]
related:
  - "[[../bkurs-be]]"
  - "[[decision-course-enrollment-model]]"
  - "[[decision-teacher-permission-matrix]]"
home_hub: bkurs-be
---

# B7A 결정 — Homework + Submission

## 결론

1. **`homework` + `homework_submission` 별도 테이블** — homework (course_id + assigned_by + title + description + due_at) + submission (homework_id + account_id + content + grade nullable + feedback).
2. **(homework, account) UNIQUE** → 제출 upsert 키. 같은 학생 같은 숙제 update 만.
3. **채점은 강사 (또는 OWNER)** — `grade` 0-100 + `feedback`. submission 의 graded_by audit.
4. **출제 권한**: 강사 본인 배정 수업 + 학원장 본인 학원 (강사 부재 대체).

## 왜 필요한가

- 학원 운영의 일상 — 숙제 출제 / 제출 / 채점이 학생-강사 핵심 인터페이스.
- bkurs-fe 학생 화면의 "내 숙제" / 강사 화면의 "채점 큐" 모두 필수.
- 학사 평가 데이터 (grade) 의 시작점.

## 안 하면 뭐가 문제

- 숙제 모델 없으면 학원 운영의 일상 미완성.
- 채점 audit (graded_by + graded_at) 없으면 분쟁 대응 불가.
- upsert 없으면 학생이 재제출 시 row 중복.

## 4 대안 비교 — submission 모델

### A. submission 별도 테이블 (homework 1:N) — **채택**

- 장점: 정규화. 학생별 / 숙제별 양방향 query.
- 단점: row 비대 (학원 × 수업 × 숙제 × 학생).

### B. submission 이 enrollment FK

- 단점: drop 후 제출 이력 소실.
- **기각**.

### C. submission JSON in homework

- 단점: 인덱스 / 채점 query 어려움.
- **기각**.

### D. submission 없이 grade 만 homework 에 array

- 단점: 채점 audit + 시점 추적 불가.
- **기각**.

## 4 대안 비교 — 채점 권한

### a. 강사 본인 배정 수업의 숙제만 채점 — **채택**

- 장점: 명확. 본인 출제 또는 본인 배정 수업.

### b. OWNER 도 본인 학원 숙제 채점 가능

- **채택 (보강)**: 강사 부재 대체. graded_by audit 으로 구분.

### c. STUDENT 자가 채점 (peer review)

- **기각**: 신뢰성 X.

## 테이블 설계

### `homework`

```sql
CREATE TABLE homework (
  id                    BIGINT AUTO_INCREMENT PRIMARY KEY,
  course_id             BIGINT NOT NULL,
  assigned_by_account_id BIGINT NOT NULL,
  title                 VARCHAR(200) NOT NULL,
  description           TEXT NULL,
  due_at                DATETIME(6) NULL,
  created_at            DATETIME(6) NOT NULL,
  updated_at            DATETIME(6) NOT NULL,
  CONSTRAINT fk_homework_course FOREIGN KEY (course_id) REFERENCES course(id) ON DELETE CASCADE,
  CONSTRAINT fk_homework_author FOREIGN KEY (assigned_by_account_id) REFERENCES account(id),
  KEY idx_homework_course_due (course_id, due_at)
);
```

### `homework_submission`

```sql
CREATE TABLE homework_submission (
  id             BIGINT AUTO_INCREMENT PRIMARY KEY,
  homework_id    BIGINT NOT NULL,
  account_id     BIGINT NOT NULL,
  content        TEXT NOT NULL,
  submitted_at   DATETIME(6) NOT NULL,
  grade          INT NULL,
  feedback       TEXT NULL,
  graded_at      DATETIME(6) NULL,
  graded_by_account_id BIGINT NULL,
  CONSTRAINT fk_submission_homework FOREIGN KEY (homework_id) REFERENCES homework(id) ON DELETE CASCADE,
  CONSTRAINT fk_submission_account FOREIGN KEY (account_id) REFERENCES account(id),
  CONSTRAINT uk_submission UNIQUE (homework_id, account_id),
  KEY idx_submission_account (account_id),
  KEY idx_submission_homework_graded (homework_id, graded_at)
);
```

## endpoint 설계

| method | path | role | 검증 |
|---|---|---|---|
| POST | /api/v1/teachers/me/courses/{cid}/homework | TEACHER | 본인 배정 |
| GET | /api/v1/teachers/me/courses/{cid}/homework | TEACHER | 본인 배정 |
| PATCH | /api/v1/teachers/me/homework/{id} | TEACHER | 본인 출제 또는 본인 배정 수업 |
| DELETE | /api/v1/teachers/me/homework/{id} | TEACHER | 본인 출제 또는 본인 배정 수업 |
| GET | /api/v1/teachers/me/homework/{id}/submissions | TEACHER | 본인 배정 수업 숙제 |
| POST | /api/v1/teachers/me/submissions/{id}/grade | TEACHER | 본인 배정 수업 숙제 |
| GET | /api/v1/students/me/homework | STUDENT | 본인 ACTIVE 수강 수업 숙제 |
| POST | /api/v1/students/me/homework/{id}/submit | STUDENT | 본인 수강 수업 숙제 |
| GET | /api/v1/students/me/submissions | STUDENT | 본인 제출 이력 |

## 회귀 라인

`HomeworkApiTest` 10종:
1. TEACHER 출제 → 201
2. STUDENT 출제 → 403
3. TEACHER 본인 배정 수업 숙제 목록
4. STUDENT 본인 수강 숙제 목록
5. STUDENT 제출 → submission row + content 보존
6. STUDENT 중복 제출 → upsert update (1 row)
7. TEACHER 채점 → grade + feedback + graded_by 매칭
8. 다른 학원 강사 출제 → 403
9. DELETE homework → submission CASCADE
10. 다른 학원 학생 숙제 조회 → 빈 목록

## 관련

- [[decision-course-enrollment-model]] — Course (homework.course_id FK)
- [[decision-teacher-permission-matrix]] — 강사/학원장 권한 패턴 재사용
- 후속: B7B (자소서 / 예상질문 / 기출 / 모의면접 / 생기부 ...)
