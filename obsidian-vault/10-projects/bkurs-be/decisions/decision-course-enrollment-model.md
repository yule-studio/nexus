---
title: "B6A 결정 — Course + Enrollment 모델 + 강사 배정 1:1 + max_students 정책"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T11:00:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, course, enrollment, b6a]
related:
  - "[[../bkurs-be]]"
  - "[[decision-teacher-invite-assignment]]"
  - "[[decision-academy-connect-membership-flow]]"
home_hub: bkurs-be
---

# B6A 결정 — Course + Enrollment

## 결론

1. **`course` + `course_enrollment` 별도 테이블** — course (academy_id + teacher_assignment_id + name + max_students) + enrollment (course_id + account_id + status ACTIVE/DROPPED).
2. **강사 배정 1:1** (course 1개에 강사 1명) — B6A 정책. 다중 강사는 B7+.
3. **max_students 정책**: ACTIVE enrollment 수가 max 도달 시 신규 enroll 409.
4. **권한 매트릭스**: OWNER (CRUD), TEACHER (본인 배정 수업 + 수강생), STUDENT (본인 학원 수업 목록 + enroll/drop).

## 왜 필요한가

- 학원 운영의 핵심 단위 — "수업" 이 없으면 시간표/출결/콘텐츠 모두 불가.
- 강사 배정 / 수강생 관리 / 정원 제한 → 학원 운영 도구 baseline.
- bkurs-fe 의 학생/강사/학원장 화면 분기에 필수.

## 안 하면 뭐가 문제

- 수업 모델 없으면 시간표 (B6B) / 출결 / 학습 콘텐츠 모두 baseline 부재.
- 정원 제한 없으면 학원 운영 어려움 (강사 1명당 학생 수 제어 불가).
- 권한 분기 없으면 다른 학원 수업 노출 / 강사가 다른 강사 수업 변경 가능.

## 4 대안 비교 — 모델

### A. course 컬럼에 enrolled_student_ids JSON — 기각

- 단점: array growth. 조회 / 인덱스 어려움.
- **기각**.

### B. course + course_enrollment 별도 테이블 (정규화) — **채택**

- 장점: 정규화 + 인덱스. "이 학생 수강 목록" / "이 수업 수강생" 양방향 query.
- 단점: row 비대 (수업 100 × 수강 30 = 3000 enrollment/학원) — 인덱스로 해결.
- **채택**.

### C. course-enrollment N:M 직접 join 테이블 (단순 association)

- 단점: status 컬럼 없으면 DROPPED 이력 보존 불가.
- **기각**: enrollment 가 단순 join 이 아닌 도메인 row.

### D. enrollment 가 별도 entity 가 아닌 inline

- **기각**: B 와 같지만 명시.

## 4 대안 비교 — 강사 배정 방식

### a. course.teacher_assignment_id 컬럼 (1:1) — **B6A 채택**

- 장점: 단순. 강사 변경 = 컬럼 update.
- 단점: 다중 강사 (메인 + 보조) 불가.
- **B6A 채택**: MVP 정책. 다중 강사는 B7+ course_teacher join 테이블.

### b. course_teacher join 테이블 (N:M)

- 단점: B6A 범위 초과. 단순성 우선.
- **결정**: B7+ 도입.

### c. 강사 배정 없는 수업 (학원장이 직접)

- 단점: 강사 배정 의미 부재. 학원장이 강사 역할.
- **B6A 정책**: teacher_assignment_id nullable — 학원장이 직접 운영하는 수업도 허용.

## 테이블 설계

### `course`

```sql
CREATE TABLE course (
  id                     BIGINT AUTO_INCREMENT PRIMARY KEY,
  academy_id             BIGINT NOT NULL,
  teacher_assignment_id  BIGINT NULL,             -- 학원장 직접 운영 시 NULL
  name                   VARCHAR(200) NOT NULL,
  description            TEXT NULL,
  max_students           INT NOT NULL DEFAULT 30,
  created_at             DATETIME(6) NOT NULL,
  updated_at             DATETIME(6) NOT NULL,
  CONSTRAINT fk_course_academy FOREIGN KEY (academy_id) REFERENCES academy(id) ON DELETE CASCADE,
  CONSTRAINT fk_course_teacher FOREIGN KEY (teacher_assignment_id) REFERENCES teacher_academy_assignment(id),
  KEY idx_course_academy (academy_id),
  KEY idx_course_teacher (teacher_assignment_id)
);
```

### `course_enrollment`

```sql
CREATE TABLE course_enrollment (
  id            BIGINT AUTO_INCREMENT PRIMARY KEY,
  course_id     BIGINT NOT NULL,
  account_id    BIGINT NOT NULL,
  status        VARCHAR(32) NOT NULL,            -- ACTIVE / DROPPED
  enrolled_at   DATETIME(6) NOT NULL,
  dropped_at    DATETIME(6) NULL,
  CONSTRAINT fk_enrollment_course FOREIGN KEY (course_id) REFERENCES course(id) ON DELETE CASCADE,
  CONSTRAINT fk_enrollment_account FOREIGN KEY (account_id) REFERENCES account(id),
  KEY idx_enrollment_course_status (course_id, status),
  KEY idx_enrollment_account_status (account_id, status)
);
```

## endpoint 설계

| method | path | role | 검증 |
|---|---|---|---|
| POST | /api/v1/academies/me/courses | OWNER | 본인 학원, teacher_assignment_id 본인 학원 강사 |
| GET | /api/v1/academies/me/courses | OWNER | 본인 학원 |
| PATCH | /api/v1/academies/me/courses/{id} | OWNER | 본인 학원 |
| DELETE | /api/v1/academies/me/courses/{id} | OWNER | 본인 학원 |
| GET | /api/v1/academies/me/courses/{id}/enrollments | OWNER | 본인 학원 |
| GET | /api/v1/teachers/me/courses | TEACHER | 본인 ACTIVE assignment 의 수업 |
| GET | /api/v1/academies/my/courses | STUDENT | 본인 학원 |
| POST | /api/v1/academies/my/courses/{id}/enroll | STUDENT | 본인 학원 수업 + max 미달 + 중복 X |
| POST | /api/v1/academies/my/courses/{id}/drop | STUDENT | 본인 ACTIVE enrollment |

## 회귀 라인

`CourseApiTest`:
1. OWNER 수업 생성 → 201
2. STUDENT 수업 생성 → 403
3. OWNER 본인 학원 수업 목록 → 200
4. TEACHER 본인 배정 수업 목록 → 200
5. STUDENT 본인 학원 수업 목록 → 200
6. STUDENT enroll → 201 ACTIVE
7. STUDENT 중복 enroll → 409
8. max_students 초과 → 409
9. STUDENT drop → DROPPED
10. 다른 학원 수업 enroll 시도 → 403 (본인 학원만)

## 관련

- [[decision-teacher-invite-assignment]] — 강사 assignment (course.teacher_assignment_id)
- [[decision-academy-connect-membership-flow]] — 학생 membership (학원장 ownership 검증)
- 후속: B6B (CourseSession 시간표 + 출결), B6C (강사 권한 매트릭스)
