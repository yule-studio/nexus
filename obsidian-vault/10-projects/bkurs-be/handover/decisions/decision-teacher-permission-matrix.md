---
title: "B6C 결정 — 강사 권한 매트릭스 정밀화 (학생 조회 / 수업 정보 일부 수정 / OWNER 출결 path)"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T12:00:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, teacher-permission, b6c]
related:
  - "[[../bkurs-be]]"
  - "[[decision-course-enrollment-model]]"
  - "[[decision-session-attendance-model]]"
  - "[[decision-student-profile-school-grade]]"
home_hub: bkurs-be
---

# B6C 결정 — 강사 권한 매트릭스 정밀화

## 결론

1. **강사 학생 프로필 조회** — 본인 배정 수업의 ACTIVE 수강생만. `StudentProfileService.getByTeacher` 신규.
2. **강사 수업 일부 수정 (`Course.updateLimited`)** — `description` / 본인이 모르는 정보 X. `name`/`maxStudents`/`teacherAssignmentId` 는 OWNER 만 (관리자 권한).
3. **OWNER 출결 입력 path 추가** — `/api/v1/academies/me/sessions/{sid}/attendance`. B6B 의 service 가 이미 OWNER 허용 → controller 추가.
4. **강사가 본인 수업 수강생 학생 프로필 GET** — 운영상 필요 (학생 이름 / 학년 / 연락처 → 학사 안내).

## 왜 필요한가

- 강사가 학생 정보 (학년 / 비상연락처) 못 보면 학사 운영 불가.
- 강사가 본인 수업 description 도 수정 못 하면 매번 학원장 요청 → 운영 부담.
- 강사 부재 시 OWNER 가 출결 입력 — service 는 허용하지만 controller path 없음.

## 안 하면 뭐가 문제

- 강사가 학생 정보 매번 학원장에게 물어봄 → 운영 부담 폭증.
- 강사 수업 정보 수정 = 학원장 요청 큐 비대화.
- OWNER 출결 입력 path 부재 → 강사 부재 시 운영 마비 (DB 직접 변경).

## 4 대안 비교 — 강사 학생 조회 범위

### A. 강사 본인 배정 수업의 ACTIVE 수강생만 — **채택**

- 장점: 명확한 범위 + ownership 검증. 학생 정보 노출 최소.
- 단점: 학원장은 모든 학생 가능 (B5) — 강사는 일부.

### B. 학원 전체 학생 (강사는 학원 소속만)

- 단점: 강사가 다른 수업 학생까지 봐야 할 필요 없음. 정보 노출 과다.
- **기각**.

### C. 강사 본인 직접 학생 등록 / 매핑

- 단점: enrollment 와 분리된 별도 매핑 = 데이터 중복.
- **기각**.

### D. 학원장이 강사별 학생 화이트리스트 관리

- 단점: 운영 부담 폭증.
- **기각**.

## 4 대안 비교 — 강사 수업 정보 수정 범위

### a. 강사 모든 컬럼 수정 가능

- 단점: name / maxStudents 변경 = 학원장 권한 침해.
- **기각**.

### b. 강사는 `description` / `room` (CourseSession) 만 — **채택**

- 장점: 학원장 관리 권한 (name/max/teacher 배정) 보존. 강사 일상 운영만.
- **채택**.

### c. 강사는 수정 불가 (학원장 요청만)

- 단점: 운영 마비 — 강사 운영 자율성 X.
- **기각**.

## endpoint 설계 (신규)

| method | path | role | 검증 |
|---|---|---|---|
| POST | /api/v1/academies/me/sessions/{sid}/attendance | OWNER | 본인 학원 session |
| PATCH | /api/v1/teachers/me/courses/{cid} | TEACHER | 본인 배정 + 제한 필드 (description) |
| GET | /api/v1/teachers/me/students/{accountId}/profile | TEACHER | 본인 배정 수업의 ACTIVE 수강생만 |

## 회귀 라인

`TeacherPermissionApiTest`:
1. OWNER 출결 입력 (POST /academies/me/sessions/.../attendance) → SUCCESS + recorded_by=owner
2. OWNER 가 강사 path (TeacherSessionAttendanceController) 호출 → 403 (TEACHER 전용)
3. TEACHER 본인 배정 수업 description PATCH → 200
4. TEACHER name 변경 시도 → 무시 (description 만 적용)
5. TEACHER maxStudents 변경 시도 → 무시
6. TEACHER 다른 학원 수업 PATCH → 403
7. TEACHER 본인 배정 수업 수강생 학생 프로필 GET → 200
8. TEACHER 본인 배정 수업 비수강 학생 프로필 GET → 403
9. TEACHER 다른 학원 수업 수강생 → 403
10. STUDENT TEACHER path 호출 → 403

## 관련

- [[decision-course-enrollment-model]] — 강사 배정 (course.teacher_assignment_id)
- [[decision-session-attendance-model]] — 출결 service 는 이미 OWNER 허용 (path 만 추가)
- [[decision-student-profile-school-grade]] — 학원장 조회 → 강사 확장
