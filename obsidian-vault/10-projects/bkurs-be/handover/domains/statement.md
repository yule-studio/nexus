---
title: "도메인 — statement (자기소개서 / Feedback)"
kind: handover
status: stable
created_at: 2026-05-21T23:48:00+09:00
tags: [bkurs-be, handover, domain, statement]
related:
  - "[[../../bkurs-be]]"
  - "[[../decisions/decision-personal-statement-model]]"
home_hub: "[[../../bkurs-be]]"
---

# statement 도메인

> 학생 자소서 (`PersonalStatement`) — DRAFT / SUBMITTED 두 상태. 강사가 댓글 형태 feedback 추가.
> DRAFT 중에는 학생이 자유롭게 수정, SUBMITTED 후에는 invariant 으로 잠금.

## 1. 책임

- 학생 본인 자소서 CRUD + submit (DRAFT → SUBMITTED 일방향)
- 강사 (해당 학생을 가르치는 수업의 강사) 의 feedback 추가 / 목록

## 2. 핵심 엔티티

| 엔티티 | 핵심 |
| --- | --- |
| `PersonalStatement` | id / studentAccountId / title / content / status (DRAFT/SUBMITTED) / submittedAt — `submit()` invariant: 이미 SUBMITTED 면 throw |
| `StatementFeedback` | id / statementId / teacherAccountId / comment / createdAt |
| `StatementStatus` | DRAFT / SUBMITTED |

## 3. 패키지 구성

```
statement/
├── controller/
│   ├── StudentStatementController  # /api/v1/students/me/personal-statements
│   └── TeacherStatementController  # /api/v1/teachers/me — 본인 수업 학생 자소서 / feedback
├── service/                        # StatementService / FeedbackService
├── domain/
├── dto/{request,response}/
├── enums/                          # StatementErrorCode (STA-)
├── exception/
└── handler/
```

## 4. API endpoint

### STUDENT (5)

| Method | Path | 책임 |
| --- | --- | --- |
| POST | `/api/v1/students/me/personal-statements` | 자소서 생성 (DRAFT) |
| GET | `/api/v1/students/me/personal-statements` | 본인 목록 |
| PATCH | `/api/v1/students/me/personal-statements/{id}` | 수정 (DRAFT 만) |
| POST | `/api/v1/students/me/personal-statements/{id}/submit` | DRAFT → SUBMITTED |
| GET | `/api/v1/students/me/personal-statements/{id}/feedback` | 받은 feedback |

### TEACHER (4)

| Method | Path | 책임 |
| --- | --- | --- |
| GET | `/api/v1/teachers/me/students/{studentId}/personal-statements` | 학생 자소서 목록 |
| GET | `/api/v1/teachers/me/personal-statements/{id}` | 상세 |
| POST | `/api/v1/teachers/me/personal-statements/{id}/feedback` | feedback 추가 |
| GET | `/api/v1/teachers/me/personal-statements/{id}/feedback` | feedback 목록 |

## 5. Migration

| V | 내용 |
| --- | --- |
| V15 | `personal_statement` (status enum) + `statement_feedback` |

## 6. 핵심 코드 흐름

### submit (일방향)

```java
public void submit() {
    if (this.status == SUBMITTED) {
        throw new StatementAlreadySubmittedException(...);
    }
    this.status = SUBMITTED;
    this.submittedAt = OffsetDateTime.now();
}
```

DRAFT 중에는 update 가능, SUBMITTED 이후엔 모든 update endpoint 가 사전 검증으로 거부.

### feedback (강사 ownership)

`FeedbackService.add(statementId, teacherId, comment)`:

```
1. statement = findById(statementId)
2. studentId = statement.studentAccountId
3. courseService.existsActiveEnrollmentTaughtBy(studentId, teacherId)   ─ 본인 수업의 학생인지
4. INSERT StatementFeedback
```

## 7. 의존

- **의존**: auth, course (학생 강사 매핑)
- **의존받음**: record (자소서 첨삭 결과를 학생부 SUBJECT 카테고리로 — 후속, 현재 자동 bridge 없음)

## 서비스 로직 (`statement/service/`)

2 service.

| 클래스 | Tx | 책임 |
| --- | --- | --- |
| `StatementService` | `@Transactional` (write) / `readOnly` (조회) | 학생 DRAFT/SUBMITTED 흐름 + ownership |
| `FeedbackService` | `@Transactional` | 강사 feedback create + role 별 listing |

### 핵심 메서드

```java
// StatementService
public PersonalStatement createMine(Long studentAccountId, String title, String content)   // status=DRAFT
public List<PersonalStatement> listMine(Long studentAccountId)
public PersonalStatement updateMine(Long studentAccountId, Long statementId, String title, String content)
                                                          // status=DRAFT 만 허용 — SUBMITTED 면 throw
public PersonalStatement submitMine(Long studentAccountId, Long statementId)
                                                          // statement.submit() — invariant: 이미 SUBMITTED 면 throw, 아니면 status SUBMITTED + submittedAt
public List<PersonalStatement> listByTeacherStudent(Long teacherAccountId, Long studentAccountId)
                                                          // course 강사-학생 매핑 검증
public PersonalStatement loadMine(Long studentAccountId, Long statementId)   // ownership 검증 helper

// FeedbackService
public StatementFeedback create(Long teacherAccountId, Long statementId, String content)
                                                          // 강사 본인 수업 학생인지 검증 + INSERT
public List<StatementFeedback> listByOwner(Long studentAccountId, Long statementId)
public List<StatementFeedback> listByTeacher(Long teacherAccountId, Long statementId)
```

## cross-link

- [[course]]
- [[record]]
- [[../decisions/decision-personal-statement-model]]
