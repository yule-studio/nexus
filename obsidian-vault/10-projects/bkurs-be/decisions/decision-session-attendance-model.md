---
title: "B6B 결정 — CourseSession + Attendance 모델 + 출결 upsert"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T11:30:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, session, attendance, b6b]
related:
  - "[[../bkurs-be]]"
  - "[[decision-course-enrollment-model]]"
home_hub: bkurs-be
---

# B6B 결정 — CourseSession + Attendance

## 결론

1. **`course_session` + `attendance` 별도 테이블** — session (course_id + start/end + room) + attendance (session_id + account_id + status).
2. **attendance status enum**: PRESENT / ABSENT / LATE / EXCUSED.
3. **(session_id, account_id) UNIQUE** → upsert 키. 같은 학생 같은 session 에 두 번 입력 = update.
4. **권한**: OWNER (session CRUD), TEACHER (본인 배정 수업 일정 + 출결 입력), STUDENT (본인 수강 수업 일정 + 본인 출결).

## 왜 필요한가

- 수업이 단순 catalog 가 아닌 **실제 운영 단위** — 시간표 / 출결 / 학습 흐름.
- 출결 추적은 학원 운영의 핵심 데이터 (학사 보고 / 결제 정산 / 학부모 알림).
- 강사가 본인 수업의 출결만 입력 — 보안 + 책임 분리.

## 안 하면 뭐가 문제

- 시간표 없으면 학생 화면 (수업 시간 표시) 동작 불가.
- 출결 없으면 학원 운영 데이터 부재 (학부모 요청 응답 못 함).
- 권한 분기 없으면 다른 강사가 본인 수업 출결 조작 가능.

## 4 대안 비교 — Attendance 모델

### A. course_session JSON 컬럼에 attendances — 기각

- 단점: array 비대 + 인덱스 어려움 + status 검색 어려움.
- **기각**.

### B. attendance 별도 테이블 (session 1:N) — **채택**

- 장점: 정규화. status 별 인덱스. 학생별 / session 별 양방향 query.
- 단점: row 비대 (학원 1 × 수업 100 × session 50 × 학생 30 = 150k rows/학원/년).
- **채택 사유**: 인덱스로 해결 + 분기 필요.

### C. attendance 가 enrollment 와 연결 (course_enrollment_id FK)

- 단점: drop 후 출결 이력 소실 (enrollment row 같이 사라짐).
- **기각**: drop 후에도 출결 보존 필요.

### D. attendance row 없으면 자동 ABSENT 가정

- 단점: 출결 입력 안 한 강사 = 자동 결석 처리 → 운영 의도 X.
- **기각**: 명시 입력만 status.

## 4 대안 비교 — 출결 입력 권한

### a. TEACHER 만 (본인 배정 수업) — **채택**

- 장점: 책임 명확. 학원장이 출결 입력하지 않음 (TEACHER 의 책임).
- 단점: 강사 부재 시 학원장이 대체 입력 못 함.
- **채택 (보강)**: OWNER 도 본인 학원 수업 출결 입력 허용 (강사 부재 시).

### b. OWNER + TEACHER (본인 학원 / 배정 수업)

- 장점: 강사 부재 시 학원장 대체 입력.
- 단점: 책임 모호 (누가 입력했는가). recorded_by_account_id 로 추적.
- **결정 (보강)**: 권한은 OWNER + TEACHER 둘 다, recorded_by 로 audit.

### c. STUDENT 본인 입력 가능 (자기 출석 신고)

- 단점: 출결 자가 보고는 신뢰성 X.
- **기각**.

## 테이블 설계

### `course_session`

```sql
CREATE TABLE course_session (
  id          BIGINT AUTO_INCREMENT PRIMARY KEY,
  course_id   BIGINT NOT NULL,
  start_at    DATETIME(6) NOT NULL,
  end_at      DATETIME(6) NOT NULL,
  room        VARCHAR(100) NULL,
  created_at  DATETIME(6) NOT NULL,
  updated_at  DATETIME(6) NOT NULL,
  CONSTRAINT fk_session_course FOREIGN KEY (course_id) REFERENCES course(id) ON DELETE CASCADE,
  KEY idx_session_course_start (course_id, start_at)
);
```

### `attendance`

```sql
CREATE TABLE attendance (
  id            BIGINT AUTO_INCREMENT PRIMARY KEY,
  session_id    BIGINT NOT NULL,
  account_id    BIGINT NOT NULL,
  status        VARCHAR(32) NOT NULL,           -- PRESENT/ABSENT/LATE/EXCUSED
  recorded_at   DATETIME(6) NOT NULL,
  recorded_by_account_id BIGINT NOT NULL,
  CONSTRAINT fk_attendance_session FOREIGN KEY (session_id) REFERENCES course_session(id) ON DELETE CASCADE,
  CONSTRAINT fk_attendance_account FOREIGN KEY (account_id) REFERENCES account(id),
  CONSTRAINT uk_attendance UNIQUE (session_id, account_id),
  KEY idx_attendance_account (account_id),
  KEY idx_attendance_session_status (session_id, status)
);
```

## endpoint 설계

| method | path | role | 검증 |
|---|---|---|---|
| POST | /api/v1/academies/me/courses/{cid}/sessions | OWNER | 본인 학원 + end > start |
| GET | /api/v1/academies/me/courses/{cid}/sessions | OWNER | 본인 학원 |
| PATCH | /api/v1/academies/me/sessions/{sid} | OWNER | 본인 학원 |
| DELETE | /api/v1/academies/me/sessions/{sid} | OWNER | 본인 학원 |
| GET | /api/v1/teachers/me/courses/{cid}/sessions | TEACHER | 본인 배정 수업 |
| POST | /api/v1/teachers/me/sessions/{sid}/attendance | TEACHER+OWNER | 본인 배정 수업 + ACTIVE 강사 / OWNER 본인 학원 |
| GET | /api/v1/teachers/me/sessions/{sid}/attendance | TEACHER | 본인 배정 수업 |
| GET | /api/v1/students/me/sessions | STUDENT | 본인 ACTIVE enrollment 수업 일정 |
| GET | /api/v1/students/me/attendance | STUDENT | 본인 출결 이력 |

## 회귀 라인

`SessionAttendanceApiTest`:
1. OWNER session 생성 → 201
2. session 시간 invariant (end <= start) → 400
3. STUDENT session 생성 → 403
4. OWNER 본인 학원 session 조회
5. TEACHER 본인 배정 수업 session 조회
6. TEACHER 출결 입력 (PRESENT) → 201
7. TEACHER 출결 두 번째 입력 (LATE) → upsert update (1 row 유지)
8. 다른 학원 강사 출결 입력 → 403
9. STUDENT 본인 출결 조회 → 본인 row 만
10. DELETE session → attendance CASCADE

## 관련

- [[decision-course-enrollment-model]] — Course (session.course_id FK 기반)
- 후속: B6C (강사 권한 매트릭스 정밀화), B7+ (학습 콘텐츠)
