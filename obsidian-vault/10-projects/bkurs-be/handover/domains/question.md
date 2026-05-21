---
title: "도메인 — question (예상/기출 문항 + Attempt)"
kind: handover
status: stable
created_at: 2026-05-21T23:49:00+09:00
tags: [bkurs-be, handover, domain, question]
related:
  - "[[../../bkurs-be]]"
  - "[[admission]]"
  - "[[../decisions/decision-question-attempt-model]]"
home_hub: "[[../../bkurs-be]]"
---

# question 도메인

> 학원 강사가 수업 단위로 발행하는 예상/기출 문항. **admission 도메인 (master 입시 문항) 과 구분
> 필요**. type 단일 enum (`EXPECTED` / `PAST`) 으로 통합 (B7C).

## 1. 책임

- 강사 Question CRUD (type=EXPECTED 또는 PAST)
- 학생 attempt — uk upsert + score 0~100 (강사 채점)

## 2. 핵심 엔티티

| 엔티티 | 핵심 |
| --- | --- |
| `Question` | id / courseId / type (EXPECTED/PAST) / year (nullable) / subject / body / answer (강사 노트, optional) |
| `QuestionAttempt` | uk(question_id, student_account_id) + answer / score / feedback / submittedAt / scoredAt |
| `QuestionType` | EXPECTED / PAST |

## 3. 패키지 구성

```
question/
├── controller/
│   ├── TeacherQuestionController  # /api/v1/teachers/me — Q CRUD + 채점
│   └── StudentQuestionController  # /api/v1/students/me — Q 응답
├── service/                       # QuestionService / AttemptService
├── domain/
├── dto/{request,response}/
├── enums/                         # QuestionErrorCode (QUE-)
├── exception/
└── handler/
```

## 4. API endpoint

### TEACHER (6)

| Method | Path | 책임 |
| --- | --- | --- |
| POST | `/api/v1/teachers/me/courses/{courseId}/questions` | 출제 |
| GET | `/api/v1/teachers/me/courses/{courseId}/questions` | 수업 Q 목록 |
| PATCH/DELETE | `/api/v1/teachers/me/questions/{id}` | 수정/삭제 |
| GET | `/api/v1/teachers/me/questions/{id}/attempts` | 응답 목록 |
| POST | `/api/v1/teachers/me/attempts/{id}/score` | 채점 |

### STUDENT (3)

| Method | Path | 책임 |
| --- | --- | --- |
| GET | `/api/v1/students/me/questions` | 내 수업 Q 목록 |
| POST | `/api/v1/students/me/questions/{id}/attempt` | 응답 (uk upsert) |
| GET | `/api/v1/students/me/attempts` | 내 attempt 목록 |

## 5. Migration

| V | 내용 |
| --- | --- |
| V16 | `question` (type enum + year nullable) + `question_attempt` (uk + score 0~100) |

## 6. admission 과 구분

| 도메인 | 발행자 | path | 용도 |
| --- | --- | --- | --- |
| `question` (본 도메인) | 학원 강사 | `/teachers/me/courses/{id}/questions` | 수업 한정 출제 |
| `admission/past_question` | MASTER | `/master/past-questions` | 입시 공통 기출 |
| `admission/major_question` | MASTER | `/master/major-questions` | 학과 면접 문항 |

학생 입장에서는 둘 다 응답 가능 — endpoint 만 다름. FE 의 high-student/past 는 admission 측을
사용.

## 서비스 로직 (`question/service/`)

2 service.

| 클래스 | Tx | 책임 |
| --- | --- | --- |
| `QuestionService` | `@Transactional` | 강사 Q CRUD |
| `AttemptService` | `@Transactional` | 학생 uk upsert + 강사 score |

### 핵심 메서드

```java
// QuestionService
public Question createByTeacher(Long teacherAccountId, Long courseId, QuestionType type, Integer year, String subject, String body, String answer)
                                                          // course.assertOwnedByTeacher → INSERT
public Question update(Long teacherAccountId, Long questionId, ...)
public void     delete(Long teacherAccountId, Long questionId)
public List<Question> listByTeacherCourse(Long teacherAccountId, Long courseId, QuestionType typeFilter)
public List<Question> listByCourses(List<Long> courseIds, QuestionType typeFilter)   // 학생용

// AttemptService
public QuestionAttempt submit(Long studentAccountId, Long questionId, String content)
                                                          // uk(question_id, student_id) upsert — 재시도 시 content 갱신
public QuestionAttempt scoreByTeacher(Long teacherAccountId, Long attemptId, Integer score, String feedback)
                                                          // 강사 권한 + score 0~100 + audit
```

## cross-link

- [[course]] — FK
- [[admission]] — 별도 입시 문항
- [[../decisions/decision-question-attempt-model]]
