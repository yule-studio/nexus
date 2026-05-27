---
title: "도메인 — course (수업 / Enrollment / Session / Attendance)"
kind: handover
status: stable
created_at: 2026-05-21T23:46:00+09:00
tags: [bkurs-be, handover, domain, course]
related:
  - "[[../../bkurs-be]]"
  - "[[../decisions/decision-course-enrollment-model]]"
  - "[[../decisions/decision-session-attendance-model]]"
  - "[[../decisions/decision-teacher-permission-matrix]]"
home_hub: "[[../../bkurs-be]]"
---

# course 도메인

> 학원 내 수업 단위. OWNER 가 수업 / Session / Attendance 의 master, TEACHER 는 limited update +
> Attendance 기록, STUDENT 는 enroll / drop + 본인 attendance 조회. 학습 도메인 (homework /
> performance / question / discussion 등) 모두 본 도메인의 Course FK 의존.

## 1. 책임

- Course (수업) — OWNER CRUD, 강사 배정 (`teacher_assignment_id` nullable), maxStudents
- CourseEnrollment — 학생 enroll / drop (ACTIVE / DROPPED) + max 검증 + 중복 방지
- CourseSession — 수업 회차 (start / end / room) — invariant end > start
- Attendance — Session 별 출결 (PRESENT / ABSENT / LATE / EXCUSED), uk(session, student) 로 upsert, recorded_by audit
- 강사 권한 매트릭스 — `Course.updateLimited(description)` 만, 학생 프로필 조회 (limited)

## 2. 핵심 엔티티

| 엔티티 | 핵심 |
| --- | --- |
| `Course` | id / academyId / teacherAssignmentId (nullable) / title / description / maxStudents — `updateLimited(desc)` invariant |
| `CourseEnrollment` | uk(courseId, studentAccountId, status=ACTIVE) — 한 학생 1 ACTIVE / course |
| `EnrollmentStatus` | ACTIVE / DROPPED |
| `CourseSession` | courseId / startAt / endAt / room — invariant end > start |
| `Attendance` | sessionId / studentAccountId + uk + recordedByAccountId (audit) |
| `AttendanceStatus` | PRESENT / ABSENT / LATE / EXCUSED |

## 3. 패키지 구성

```
course/
├── controller/                      # 9개
│   ├── OwnerCourseController        # /api/v1/academies/me/courses (CRUD + enrollments)
│   ├── OwnerSessionController       # session CRUD
│   ├── OwnerAttendanceController    # attendance upsert (학원장 직접)
│   ├── TeacherCourseController      # /api/v1/teachers/me/courses
│   ├── TeacherCourseUpdateController# limited PATCH (description 만)
│   ├── TeacherSessionAttendanceController # 강사 attendance 기록
│   ├── TeacherStudentProfileController    # 본인 수업 학생 프로필 조회
│   ├── StudentCourseController      # enroll / drop
│   └── StudentSessionAttendanceController # 본인 세션 + 본인 출결
├── service/                         # CourseService / EnrollmentService / SessionService / AttendanceService
├── domain/                          # 4 entity + enums + Repository
├── dto/{request,response}/
├── enums/                           # CourseErrorCode (CRS-)
├── exception/
└── handler/
```

## 4. API endpoint (요약 9개 group)

### OWNER (5)

| Method | Path | 책임 |
| --- | --- | --- |
| POST/GET | `/api/v1/academies/me/courses` | 수업 CRUD |
| PATCH/DELETE | `/api/v1/academies/me/courses/{id}` | |
| GET | `/api/v1/academies/me/courses/{id}/enrollments` | 수업 등록 학생 목록 |
| POST/GET/PATCH/DELETE | `/api/v1/academies/me/courses/{id}/sessions` | Session CRUD |
| POST | `/api/v1/academies/me/sessions/{id}/attendance` | 학원장 출결 직접 기록 |

### TEACHER (2)

| Method | Path | 책임 |
| --- | --- | --- |
| GET | `/api/v1/teachers/me/courses` | 본인 배정 수업 |
| GET | `/api/v1/teachers/me/courses/{id}/enrollments` | 학생 목록 |
| PATCH | `/api/v1/teachers/me/courses/{id}` | description 만 수정 (limited) |
| GET | `/api/v1/teachers/me/courses/{id}/sessions` | 세션 목록 |
| POST | `/api/v1/teachers/me/sessions/{id}/attendance` | 출결 기록 |
| GET | `/api/v1/teachers/me/sessions/{id}/attendance` | 출결 조회 |
| GET | `/api/v1/teachers/me/students/{id}/profile` | 본인 수업 학생 프로필 |

### STUDENT (3)

| Method | Path | 책임 |
| --- | --- | --- |
| GET | `/api/v1/academies/my/courses` | 본인 학원 수업 |
| POST | `/api/v1/academies/my/courses/{id}/enroll` | enroll (max 검증) |
| POST | `/api/v1/academies/my/courses/{id}/drop` | drop |
| GET | `/api/v1/students/me/sessions` | 본인 등록 수업의 세션 |
| GET | `/api/v1/students/me/attendance` | 본인 출결 |

## 5. Migration

| V | 내용 |
| --- | --- |
| V12 | `course` + `course_enrollment` (uk + status enum) |
| V13 | `course_session` (CHECK end > start) + `attendance` (uk session/student + recorded_by audit) |

## 6. 핵심 코드 흐름

### enroll (max + 중복 방지)

```
EnrollmentService.enroll(courseId, studentId)
  ▼
1. course = courseRepo.findByIdAndAcademyId(...) ─ academy 일치 검증
2. activeCount = enrollmentRepo.countByCourseIdAndStatus(courseId, ACTIVE)
3. activeCount < course.maxStudents 검증
4. existsByCourseIdAndStudentAndStatus(courseId, sid, ACTIVE) ? throw
5. INSERT (ACTIVE)
```

### attendance upsert (강사 또는 학원장)

```
AttendanceService.recordOrUpdate(sessionId, studentId, status, recordedById)
  ▼
existing = repo.findBySessionIdAndStudentAccountId(...)
  ├── 있음 → existing.updateStatus(status, now, recordedById)
  └── 없음 → new + save
```

uk 위반 동시성: try / save → uk violation → 재조회 후 update.

### 강사 권한 매트릭스 (B6C)

`Course` entity:

```java
public void updateLimited(String description) {
    this.description = description;
    this.updatedAt = OffsetDateTime.now();
}
// title / maxStudents 등 setter 없음 — OWNER 만 update*() 호출 가능
```

service: `CourseService.updateByTeacher(courseId, teacherId, desc)` — write 트랜잭션 + 강사 배정
검증 + `course.updateLimited(desc)`.

## 7. 의존

- **의존**: academy (Academy / TeacherAcademyAssignment), student (StudentProfile)
- **의존받음**: homework, statement, question, interview, performance, simulation, discussion, record, dashboard, stats

## 서비스 로직 (`course/service/`)

4 service — Course 본체 + Enrollment + Session + Attendance. 학습 도메인 (homework/PE/discussion/...) 의 권한 검증 진입점.

| 클래스 | Tx | 책임 |
| --- | --- | --- |
| `CourseService` | `@Transactional` (write) / `readOnly` (조회) | OWNER CRUD + 강사 limited update + 강사 권한 검증 helper |
| `EnrollmentService` | `@Transactional` | 학생 enroll (max + 중복 방지) / drop |
| `SessionService` | `@Transactional` | OWNER 세션 CRUD (end>start invariant) |
| `AttendanceService` | `@Transactional` (write) / `readOnly` (조회) | upsert (uk 위반 시 update) + recorded_by audit |

### 핵심 메서드

```java
// CourseService
public Course create(Long ownerAccountId, String name, String description, Long teacherAssignmentId, Integer maxStudents)
                                                          // academy 결정 + teacherAssignment 검증 (본인 학원 소속)
public Course update(Long ownerAccountId, Long courseId, ...)
public void   delete(Long ownerAccountId, Long courseId)  // FK CASCADE 로 enrollment / session 등 정리
public Course loadOwned(Long ownerAccountId, Long courseId)        // ownership 검증 후 entity 반환
public Course updateByTeacher(Long teacherAccountId, Long courseId, String description)
                                                          // teacher 배정 검증 + course.updateLimited(desc) — write TX

// EnrollmentService
public CourseEnrollment enroll(Long studentAccountId, Long courseId)
                                                          // 1. course = courseRepo.findByIdAndAcademyId (academy 일치)
                                                          // 2. activeCount < maxStudents 검증
                                                          // 3. 중복 ACTIVE 없는지 검증
                                                          // 4. INSERT ACTIVE
public CourseEnrollment drop(Long studentAccountId, Long courseId)

// SessionService
public CourseSession create(Long ownerAccountId, Long courseId, OffsetDateTime start, OffsetDateTime end, String room)
                                                          // course ownership + end>start invariant
public CourseSession loadOwned(Long ownerAccountId, Long sessionId)

// AttendanceService
public Attendance upsert(Long recordedByAccountId, Long sessionId, Long studentAccountId, AttendanceStatus status)
                                                          // existing 있으면 status / recordedBy / now 갱신, 없으면 INSERT
public List<Attendance> listMine(Long studentAccountId)
```

강사 권한 매트릭스 결정: [[../decisions/decision-teacher-permission-matrix]].

## cross-link

- [[homework]] / [[statement]] / [[question]] / [[interview]] / [[performance]] / [[simulation]] / [[discussion]] — 모두 course FK
- [[../decisions/decision-course-enrollment-model]]
- [[../decisions/decision-session-attendance-model]]
- [[../decisions/decision-teacher-permission-matrix]]
