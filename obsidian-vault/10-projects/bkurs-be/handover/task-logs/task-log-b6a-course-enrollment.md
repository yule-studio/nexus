---
title: "B6A — Course + Enrollment baseline 작업 로그"
kind: task-log
project: bkurs-be
session_id: bkurs-be-b6-course
created_at: 2026-05-20T11:00:00+09:00
agent: engineering-agent/tech-lead
status: in-progress
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-course-enrollment-model]]"
  - "[[task-log-b5-student-profile]]"
tags: [task-log, course, enrollment, b6a]
---

# 작업

GitHub issue [#25](https://github.com/seum-running/bkurs-be/issues/25) — Course + Enrollment baseline.

# 결정 요약 (인수인계용)

| 결정 | 내용 | 상세 |
|---|---|---|
| 모델 | course + course_enrollment 별도 테이블 (정규화) | [[../decisions/decision-course-enrollment-model]] |
| 강사 배정 | 1:1 (teacher_assignment_id nullable, 학원장 직접 운영 허용) | 다중 강사는 B7+ |
| 정원 | max_students INT DEFAULT 30, ACTIVE 수 검증 | race condition은 B7+ |
| 권한 | 4 role 분기 (OWNER CRUD / TEACHER 본인 배정 / STUDENT 본인 학원 enroll) | role 별 controller |
| URL prefix | /me OWNER/TEACHER vs /my STUDENT | B3C 패턴 일관 |

# 진행

## 2026-05-20

### 1. branch / issue
- B5 머지 대기 중 stacked PR — base=feature/issue-23-b5-student-profile
- issue #25 + decision 노트 (4 대안 × 2)

### 2. 코드 (commit 단위)
- C1: Course + CourseEnrollment + V12 + EnrollmentStatus enum + invariant 메서드 + Repository
- C2: 5 예외 + CourseService (OWNER CRUD + teacher validation) + EnrollmentService (학생 enroll/drop + max/중복)
- C3: 4 DTO + 3 controller (Owner/Teacher/Student) + CourseExceptionHandler
- C4: CourseApiTest 10 + CHANGELOG B6A

### 3. PR
- PR [#26](https://github.com/seum-running/bkurs-be/pull/26) (feature/issue-25 → feature/issue-23-b5-student-profile, stacked)
- 회귀 **112/112 GREEN**

# 인수인계 (Handover) — 왜 이렇게 만들었나

| 구현 | 선택 이유 | 대안 / 후속 |
|---|---|---|
| course + enrollment 별도 (정규화) | array JSON 안티패턴 회피. 양방향 query (학생 수강 목록 / 수업 수강생) 인덱스. | row 비대 (수업 100 × 30 = 3000) 인덱스로 해결. |
| 강사 배정 1:1 (nullable) | MVP 정책. 학원장 직접 운영 수업도 허용 (nullable). | 다중 강사 (메인+보조) 는 B7+ course_teacher join 테이블. |
| max_students INT DEFAULT 30 | 학원 운영 현실 적정값. 학원장이 PATCH 로 조정 가능. | 무제한 (NULL) 도 가능했지만 운영 안전성 우선. |
| count() + insert 정원 검증 | 단순. race condition 은 동시 enroll 시 위반 가능 (확률 낮음). | unique constraint 또는 select for update (B7+ scale 시점). |
| `belongsTo(academyId)` ownership 헬퍼 | service 가 academy.id 와 course.academy_id 매칭 반복 → 도메인 메서드로 응집. | 다른 도메인 (announcement / membership) 의 동일 패턴. |
| 학원장 controller 클래스 레벨 @PreAuthorize OWNER | path /me/courses 가 OWNER 전용 명확. | 메서드별 안 함. |
| 학생 controller 는 인증 사용자 누구나 (Resolver 가 검증) | 학생 외 role 가 GET 요청 시 빈 목록 (정보 노출 없음). enroll 은 service 가 학원 격리 검증. | 명시 @PreAuthorize STUDENT_* 도 가능. Resolver 패턴 일관. |
| 강사 controller 클래스 레벨 @PreAuthorize TEACHER | TEACHER 전용 path /teachers/me/courses. | OWNER 가 TEACHER 처럼 본인 배정 수업 보는 use case 없음. |
| 강사 본인 배정 수업 = ACTIVE assignment 의 수업 | 강사가 학원 떠나면 (REMOVED) 수업 노출 안 됨 (실수 보호). | 향후 강사 권한 매트릭스 (B6C) 에서 더 정밀 분기. |
| EnrollmentService 가 CourseService.getById/loadOwned 위임 | course ownership 검증 분산 회피. CourseService 가 SSoT. | service 간 호출 — package 분리 시 cross-package 의존 정상. |
| listByCourseAssignedToTeacher 의 in-memory 매칭 | 강사 수업 N 개 fetch → courseId 비교. N 작음 (강사당 수업 10 미만) → 비용 무시. | 향후 강사 수업 100+ 면 별도 query. |
| @BeforeEach cleanup 순서 (enrollment → course → 기존) | V12 FK course CASCADE academy + enrollment CASCADE course → 명시적 정리. | FK CASCADE 의존 안 해도 안전. |

# 다음 단계 (예정)

- B2A → ... → B6A 순으로 사용자 manual merge → CI 자동 실행
- B6B 시작 — CourseSession (시간표) + Attendance (출결)
- B6C — 강사 권한 매트릭스 정밀화 (강사가 출결 입력 / 학생 데이터 수정)

# 회수 / pitfall (누적)

- enroll 정원 검증 race condition — count + insert 패턴. unique constraint 또는 select for update 검토 (B7+ 대용량 시점).

# 참고

- [[../decisions/decision-course-enrollment-model]]
- [[task-log-b5-student-profile]]
