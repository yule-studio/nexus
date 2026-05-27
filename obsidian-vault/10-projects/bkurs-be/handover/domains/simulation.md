---
title: "도메인 — simulation (모의지원 + feasibility 1~5)"
kind: handover
status: stable
created_at: 2026-05-21T23:55:00+09:00
tags: [bkurs-be, handover, domain, simulation]
related:
  - "[[../../bkurs-be]]"
  - "[[record]]"
  - "[[../decisions/decision-simulation-model]]"
home_hub: "[[../../bkurs-be]]"
---

# simulation 도메인

> 강사가 모의지원 시나리오 (3 type) 발행, 학생이 응답, 강사가 feasibility (1~5 별점) review.
> **정답 없는 시나리오** — performance 와 의미 분리. record bridge → CAREER.

## 1. 책임

- 강사 SimulationCase CRUD (3 type: UNIV_APPLICATION / CAREER / FINANCIAL)
- 학생 SimulationResponse upsert (재제출 시 review 무효화)
- 강사 review (feasibility 1~5 + 코멘트) → record CAREER bridge

## 2. 핵심 엔티티

| 엔티티 | 핵심 |
| --- | --- |
| `SimulationCase` | id / courseId / type / scenario / expectedOutcome (강사 노트) |
| `SimulationResponse` | uk(case_id, student_id) + answer / feasibility (1~5, nullable) / reviewComment / reviewedBy / submittedAt / reviewedAt |
| `SimulationType` | UNIV_APPLICATION / CAREER / FINANCIAL |

## 3. 패키지 구성

```
simulation/
├── controller/
│   ├── TeacherSimulationController  # case CRUD + review
│   └── StudentSimulationController  # case 목록 + 응답
├── service/                         # SimulationService — review 시 event publish
├── domain/
├── dto/{request,response}/
├── enums/                           # SimulationErrorCode (SIM-)
├── exception/
└── handler/
```

## 4. API endpoint

### TEACHER (6)

| Method | Path | 책임 |
| --- | --- | --- |
| POST/GET | `/api/v1/teachers/me/courses/{courseId}/simulation-cases` | 시나리오 CRUD |
| PATCH/DELETE | `/api/v1/teachers/me/simulation-cases/{id}` | |
| GET | `/api/v1/teachers/me/simulation-cases/{id}/responses` | 응답 목록 |
| POST | `/api/v1/teachers/me/simulation-responses/{id}/review` | review → bridge |

### STUDENT (2)

| Method | Path | 책임 |
| --- | --- | --- |
| GET | `/api/v1/students/me/simulation-cases` | 내 수업 시나리오 |
| POST | `/api/v1/students/me/simulation-cases/{id}/respond` | 응답 (재제출 → review 무효화) |

## 5. Migration

| V | 내용 |
| --- | --- |
| V22 | `simulation_case` + `simulation_response` (uk + feasibility 1~5 CHECK + review 무효화 invariant) |

## 6. 핵심 invariant

### 재제출 review 무효화

```java
public void resubmit(String answer) {
    this.answer = answer;
    this.resubmittedAt = OffsetDateTime.now();
    this.feasibility = null;
    this.reviewComment = null;
    this.reviewedByAccountId = null;
    this.reviewedAt = null;
}
```

performance 와 동일 패턴 — 재제출은 의도적 갱신.

### review + bridge

```
SimulationService.review(responseId, teacherId, feasibility, comment)
  ▼
1. response.review(feasibility, comment, teacherId, now)
2. eventPublisher.publishEvent(new SimulationReviewedEvent(
       studentId, responseId, scenarioTitle))
       ─→ record bridge: CAREER 카테고리 upsert
```

## 7. performance 와 차이

- performance: max_score 1~1000 (정답 기준)
- simulation: feasibility 1~5 (현실성 별점 — 정답 없는 진로 시나리오)

## 8. 의존

- **의존**: course, record (event publish)
- **의존받음**: record bridge, stats, search

## 서비스 로직 (`simulation/service/`)

1 service. 재제출 시 review 무효화 + CAREER bridge.

| 클래스 | Tx | 책임 |
| --- | --- | --- |
| `SimulationService` | `@Transactional` (write) / `readOnly` (조회) | 강사 case CRUD + 학생 response (재제출=resubmit) + 강사 review (feasibility 1~5) + CAREER bridge |

### 핵심 메서드

```java
public SimulationCase createByTeacher(Long teacherAccountId, Long courseId, SimulationType type, String scenario, String expectedOutcome)
public SimulationCase updateByTeacher(Long teacherAccountId, Long caseId, ...)
public void           deleteByTeacher(Long teacherAccountId, Long caseId)
public List<SimulationCase> listByTeacherCourse(Long teacherAccountId, Long courseId)
public List<SimulationCase> listByStudent(Long studentAccountId)

public SimulationResponse respondByStudent(Long studentAccountId, Long caseId, String answer)
                                                          // existing 있으면 resubmit() — feasibility/review/reviewedBy null
                                                          // 없으면 INSERT
public List<SimulationResponse> listResponsesByTeacher(Long teacherAccountId, Long caseId)
public SimulationResponse reviewByTeacher(Long teacherAccountId, Long responseId, Integer feasibility, String comment)
                                                          // feasibility 1~5 invariant + audit
                                                          // → publishEvent(new SimulationReviewedEvent(...))
                                                          //   record bridge 가 CAREER 카테고리로 upsert (same TX)
```

## cross-link

- [[course]]
- [[record]] — CAREER bridge
- [[performance]] — 의미 분리
- [[../decisions/decision-simulation-model]]
