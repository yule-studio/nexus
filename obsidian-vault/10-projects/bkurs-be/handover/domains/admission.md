---
title: "도메인 — admission (입시 면접 문항 / Past + Major)"
kind: handover
status: stable
created_at: 2026-05-22T00:02:00+09:00
tags: [bkurs-be, handover, domain, admission]
related:
  - "[[../../bkurs-be]]"
  - "[[question]]"
home_hub: "[[../../bkurs-be]]"
---

# admission 도메인

> **MASTER (본사) 가 발행**하는 입시 공통 문항. PastQuestion (과거 출제) + MajorQuestion (학과 면접).
> 학원 강사 발행 문항인 `question` 도메인과 구분.

## 1. 책임

- MASTER: PastQuestion / MajorQuestion CRUD
- STUDENT: 본인 학년 / 학과 관련 문항 view

## 2. 핵심 엔티티

| 엔티티 | 핵심 |
| --- | --- |
| `PastQuestion` | id / year / subject / question / answer (참고) / grade (AdmissionGrade) |
| `MajorQuestion` | id / major / question / answer (참고) / grade |
| `AdmissionGrade` | 학년 enum (예: HIGH_1 / HIGH_2 / HIGH_3) |

## 3. 패키지 구성

```
admission/
├── controller/
│   ├── MasterPastQuestionController     # /api/v1/master/past-questions — MASTER CRUD
│   ├── MasterMajorQuestionController    # /api/v1/master/major-questions — MASTER CRUD
│   ├── StudentPastQuestionController    # /api/v1/students/me/past-questions — GET
│   └── StudentMajorQuestionController   # /api/v1/students/me/major-questions — GET
├── service/
├── domain/
├── dto/{request,response}/
├── enums/                               # AdmissionErrorCode (ADM-)
├── exception/
└── handler/
```

## 4. API endpoint

### MASTER (8)

| Method | Path | 책임 |
| --- | --- | --- |
| POST/GET/PATCH/DELETE | `/api/v1/master/past-questions[/{id}]` | PastQuestion CRUD |
| POST/GET/PATCH/DELETE | `/api/v1/master/major-questions[/{id}]` | MajorQuestion CRUD |

### STUDENT (2)

| Method | Path | 책임 |
| --- | --- | --- |
| GET | `/api/v1/students/me/past-questions` | 학년 / subject 필터 view |
| GET | `/api/v1/students/me/major-questions` | 본인 관심 학과 view |

## 5. Migration

| V | 내용 |
| --- | --- |
| V28 | `past_question` (PR #152) |
| V29 | `major_question` (PR #153) |

## 6. question 도메인과의 비교

| 항목 | `question` (학원 강사) | `admission` (MASTER) |
| --- | --- | --- |
| 발행자 | 강사 | MASTER (본사) |
| 범위 | 수업 단위 | 전사 공통 |
| path | `/teachers/me/...` `/students/me/questions` | `/master/...` `/students/me/{past,major}-questions` |
| 학생 응답 | 있음 (`QuestionAttempt`) | 없음 (view only) |

FE high-student/past 페이지는 admission 측을 사용 (master 발행 자료).

## 7. 의존

- **의존**: auth (role MASTER / STUDENT 검증)

## 서비스 로직 (`admission/service/`)

2 service — Past / Major 별 동일 패턴.

| 클래스 | Tx | 책임 |
| --- | --- | --- |
| `PastQuestionService` | `@Transactional` | MASTER CRUD + grade 별 listing |
| `MajorQuestionService` | `@Transactional` | MASTER CRUD + grade 별 listing |

### 핵심 메서드

```java
// PastQuestionService
public PastQuestion create(Long masterAccountId, AdmissionGrade grade, String university, String major, String admissionType, Integer year, String question, String answer)
public PastQuestion update(Long id, String university, String major, String admissionType, ...)
public void         delete(Long id)
public List<PastQuestion> listByGrade(AdmissionGrade grade)

// MajorQuestionService — 동일 패턴
public MajorQuestion create(Long masterAccountId, AdmissionGrade grade, Integer schoolYear, String departmentCode, ...)
public MajorQuestion update(Long id, Integer schoolYear, String departmentCode, ...)
public void          delete(Long id)
public List<MajorQuestion> listByGrade(AdmissionGrade grade)
```

학생 측은 query only — 별도 service 없이 controller 가 repository 직접 호출 (read-only 단순 view).

## cross-link

- [[question]] — 학원 강사 출제 (구분)
- [[auth]]
