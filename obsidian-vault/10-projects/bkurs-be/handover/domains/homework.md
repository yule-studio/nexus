---
title: "도메인 — homework (숙제 / 제출 / 채점)"
kind: handover
status: stable
created_at: 2026-05-21T23:47:00+09:00
tags: [bkurs-be, handover, domain, homework]
related:
  - "[[../../bkurs-be]]"
  - "[[../decisions/decision-homework-submission-model]]"
home_hub: "[[../../bkurs-be]]"
---

# homework 도메인

> 강사가 수업 단위로 숙제 생성, 학생이 제출 (upsert), 강사가 채점. **performance 와 의미 분리** —
> homework 는 일상 과제, performance 는 정식 수행평가 (생기부 연계).

## 1. 책임

- 강사 Homework CRUD (course 단위)
- 학생 제출 — upsert (한 숙제 1 제출, 재제출은 update)
- 강사 채점 (0~100) + audit (gradedBy / gradedAt)

## 2. 핵심 엔티티

| 엔티티 | 핵심 |
| --- | --- |
| `Homework` | id / courseId / title / description / dueAt (nullable) / assignedByAccountId |
| `HomeworkSubmission` | uk(homework_id, student_account_id) + content / grade (0~100, nullable) / feedback / gradedByAccountId / submittedAt / gradedAt |

## 3. 패키지 구성

```
homework/
├── controller/
│   ├── TeacherHomeworkController  # /api/v1/teachers/me — homework + submission CRUD
│   └── StudentHomeworkController  # /api/v1/students/me — 내 숙제 + 제출
├── service/                       # HomeworkService / SubmissionService
├── domain/                        # Homework / HomeworkSubmission + Repository
├── dto/{request,response}/
├── enums/                         # HomeworkErrorCode (HMW-)
├── exception/
└── handler/
```

## 4. API endpoint

### TEACHER (6)

| Method | Path | 책임 |
| --- | --- | --- |
| POST | `/api/v1/teachers/me/courses/{courseId}/homework` | 숙제 생성 |
| GET | `/api/v1/teachers/me/courses/{courseId}/homework` | 수업 숙제 목록 |
| PATCH | `/api/v1/teachers/me/homework/{id}` | 수정 |
| DELETE | `/api/v1/teachers/me/homework/{id}` | 삭제 |
| GET | `/api/v1/teachers/me/homework/{id}/submissions` | 제출 목록 |
| POST | `/api/v1/teachers/me/submissions/{id}/grade` | 채점 |

### STUDENT (3)

| Method | Path | 책임 |
| --- | --- | --- |
| GET | `/api/v1/students/me/homework` | 내 수업 숙제 |
| POST | `/api/v1/students/me/homework/{id}/submit` | 제출 (upsert) |
| GET | `/api/v1/students/me/submissions` | 내 제출 목록 |

## 5. Migration

| V | 내용 |
| --- | --- |
| V14 | `homework` + `homework_submission` (uk + grade 0~100 invariant + audit) |

## 6. 핵심 코드 흐름

### submit (upsert)

```
SubmissionService.upsert(homeworkId, studentId, content)
  ▼
existing = repo.findByHomeworkIdAndStudentAccountId(...)
  ├── 있음 → existing.updateContent(content, now) + grade 보존
  └── 없음 → new + save
```

uk 충돌 시 silent — 동시 클릭 안전.

### grade

```
SubmissionService.grade(submissionId, teacherId, grade, feedback)
  ▼
1. submission = repo.findById(...)
2. course / teacher 배정 검증 — 본인 수업의 숙제 제출인지
3. submission.grade(grade, feedback, teacherId, now) — gradedAt audit
```

> 강사 권한 검증: course → teacher_assignment_id 와 본인 일치. `CourseService.assertOwnedByTeacher`
> 같은 helper 가 cross-domain.

## 7. 의존

- **의존**: auth, academy, course
- **의존받음**: stats (homework 통계), dashboard (강사 pending 카운트), FE middle-student/homework

## 8. performance 와 차이

- homework: 일상 과제, 비정형, 생기부 자동 bridge 없음
- performance: 4 카테고리 정식 평가, max_score 1~1000, record 자동 bridge

[[performance]] 참조.

## 서비스 로직 (`homework/service/`)

2 service. PE 와 비교: 본 도메인은 record bridge 없음.

| 클래스 | Tx | 책임 |
| --- | --- | --- |
| `HomeworkService` | `@Transactional` | 강사 CRUD + ownership helper |
| `SubmissionService` | `@Transactional` | 학생 upsert + 강사 채점 (audit) |

### 핵심 메서드

```java
// HomeworkService
public Homework createByTeacher(Long teacherAccountId, Long courseId, String title, String description, OffsetDateTime dueAt)
                                                          // course.assertOwnedByTeacher → INSERT
public Homework update(Long teacherAccountId, Long homeworkId, ...)
public void     delete(Long teacherAccountId, Long homeworkId)
public List<Homework> listByTeacherCourse(Long teacherAccountId, Long courseId)
public List<Homework> listByCourses(List<Long> courseIds)        // 학생 본인 enrolled 수업 IDs 로 호출
public Homework loadOwnedByTeacher(Long teacherAccountId, Long homeworkId)

// SubmissionService
public HomeworkSubmission submit(Long studentAccountId, Long homeworkId, String content)
                                                          // 1. homework lookup
                                                          // 2. enrollment ACTIVE 검증
                                                          // 3. existing 있으면 content 갱신, 없으면 INSERT
public HomeworkSubmission gradeByTeacher(Long teacherAccountId, Long submissionId, Integer grade, String feedback)
                                                          // 강사 권한 + grade 0~100 invariant + audit (gradedBy / gradedAt)
public List<HomeworkSubmission> listByHomework(Long teacherAccountId, Long homeworkId)
public List<HomeworkSubmission> listMine(Long studentAccountId)
```

## cross-link

- [[course]] — FK
- [[stats]] / [[dashboard]] — 미채점 카운트
- [[performance]] — 의미 분리
- [[../decisions/decision-homework-submission-model]]
