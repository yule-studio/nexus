---
title: "도메인 — performance (수행평가 + 재제출 채점 무효화)"
kind: handover
status: stable
created_at: 2026-05-21T23:54:00+09:00
tags: [bkurs-be, handover, domain, performance]
related:
  - "[[../../bkurs-be]]"
  - "[[record]]"
  - "[[../decisions/decision-performance-evaluation-model]]"
home_hub: "[[../../bkurs-be]]"
---

# performance 도메인

> 정식 수행평가 — homework 와 의미 분리. 4 카테고리 (WRITTEN / PROJECT / PRESENTATION /
> EXPERIMENT), max_score 1~1000, **재제출 시 채점 자동 무효화**. record bridge → SUBJECT.

## 1. 책임

- 강사 PerformanceEvaluation CRUD + max_score (1~1000)
- 학생 PerformanceSubmission upsert (재제출)
- 강사 채점 (0~max_score) + record SUBJECT 자동 bridge
- 재제출 시 채점 무효화 (grade=null, gradedBy=null)

## 2. 핵심 엔티티

| 엔티티 | 핵심 |
| --- | --- |
| `PerformanceEvaluation` | id / courseId / category (4) / title / description / maxScore (1~1000) / dueAt |
| `PerformanceSubmission` | uk(eval_id, student_id) + content / grade (nullable, 0~maxScore) / feedback / gradedBy / submittedAt / resubmittedAt / gradedAt |
| `PerformanceCategory` | WRITTEN / PROJECT / PRESENTATION / EXPERIMENT |

## 3. 패키지 구성

```
performance/
├── controller/
│   ├── TeacherPerformanceController  # /api/v1/teachers/me — eval + grade
│   └── StudentPerformanceController  # /api/v1/students/me — submission
├── service/                          # PerformanceEvaluationService — grade 시 event publish
├── domain/
├── dto/{request,response}/
├── enums/                            # PerformanceErrorCode (PEV-)
├── exception/
└── handler/
```

## 4. API endpoint

### TEACHER (6)

| Method | Path | 책임 |
| --- | --- | --- |
| POST | `/api/v1/teachers/me/courses/{courseId}/performance-evaluations` | 발행 |
| GET | `/api/v1/teachers/me/courses/{courseId}/performance-evaluations` | 목록 |
| PATCH/DELETE | `/api/v1/teachers/me/performance-evaluations/{id}` | |
| GET | `/api/v1/teachers/me/performance-evaluations/{id}/submissions` | 제출 목록 |
| POST | `/api/v1/teachers/me/performance-submissions/{id}/grade` | 채점 → bridge |

### STUDENT (2)

| Method | Path | 책임 |
| --- | --- | --- |
| GET | `/api/v1/students/me/performance-evaluations` | 내 수업 평가 |
| POST | `/api/v1/students/me/performance-evaluations/{id}/submit` | 제출 (재제출 → 채점 무효화) |

## 5. Migration

| V | 내용 |
| --- | --- |
| V21 | `performance_evaluation` + `performance_submission` (uk + grade audit + max_score 1~1000 CHECK) |

## 6. 핵심 invariant

### 재제출 채점 무효화

```java
public void resubmit(String content) {
    this.content = content;
    this.resubmittedAt = OffsetDateTime.now();
    // 채점 무효화
    this.grade = null;
    this.gradedByAccountId = null;
    this.gradedAt = null;
    this.feedback = null;
}
```

이유: 재제출은 학생이 의도적으로 갱신한 답안 — 이전 grade 는 무효. 강사가 다시 채점해야 함.

### grade 시 bridge

```
PerformanceEvaluationService.grade(submissionId, teacherId, grade, feedback)
  ▼
1. submission.grade(grade, feedback, teacherId, now)
2. eventPublisher.publishEvent(new PerformanceGradedEvent(
       studentId, submissionId, evaluation.title, grade, evaluation.maxScore))
       ─→ record bridge: SUBJECT 카테고리 upsert
```

기존 record entry 가 있으면 overwrite (uk source_type=PERFORMANCE, source_id=submissionId).

## 7. homework / record 와 의미 분리

| 도메인 | 무게 | 생기부 bridge |
| --- | --- | --- |
| homework | 일상 과제 | 없음 |
| performance | 정식 평가 | SUBJECT 자동 |
| record (MANUAL) | 강사 직접 입력 | (자체) |

## 8. 의존

- **의존**: course, record (event publish)
- **의존받음**: record bridge, stats, dashboard

## 서비스 로직 (`performance/service/`)

1 service. **재제출 시 채점 자동 무효화** + SUBJECT bridge.

| 클래스 | Tx | 책임 |
| --- | --- | --- |
| `PerformanceEvaluationService` | `@Transactional` (write) / `readOnly` (조회) | 강사 PE CRUD + 학생 submit (재제출=resubmit) + 강사 grade + SUBJECT bridge |

### 핵심 메서드

```java
public PerformanceEvaluation createByTeacher(Long teacherAccountId, Long courseId, PerformanceCategory category, String title, String description, Integer maxScore, OffsetDateTime dueAt)
                                                          // maxScore 1~1000 invariant
public PerformanceEvaluation updateByTeacher(Long teacherAccountId, Long evalId, String title, ...)
public void                  deleteByTeacher(Long teacherAccountId, Long evalId)
public List<PerformanceEvaluation> listByTeacherCourse(Long teacherAccountId, Long courseId)
public List<PerformanceEvaluation> listByStudent(Long studentAccountId)

public PerformanceSubmission submitByStudent(Long studentAccountId, Long evalId, String content)
                                                          // existing 있으면 resubmit() — grade/gradedBy/feedback null 처리
                                                          // 없으면 INSERT
public List<PerformanceSubmission> listSubmissionsByTeacher(Long teacherAccountId, Long evalId)
public PerformanceSubmission gradeByTeacher(Long teacherAccountId, Long submissionId, Integer grade, String feedback)
                                                          // grade 0~maxScore invariant + audit
                                                          // → publishEvent(new PerformanceGradedEvent(...))
                                                          //   record bridge 가 SUBJECT 카테고리로 upsert (same TX)
```

재제출 무효화 패턴은 [[simulation]] 과 동일.

## cross-link

- [[course]]
- [[record]] — SUBJECT bridge
- [[homework]] — 의미 차이
- [[../decisions/decision-performance-evaluation-model]]
