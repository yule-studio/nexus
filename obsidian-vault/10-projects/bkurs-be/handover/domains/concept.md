---
title: "도메인 — concept (진로 / 컨셉 진단)"
kind: handover
status: stable
created_at: 2026-05-21T23:59:00+09:00
tags: [bkurs-be, handover, domain, concept]
home_hub: "[[../../bkurs-be]]"
related:
  - "[[../../bkurs-be]]"
home_hub: "[[../../bkurs-be]]"
---

# concept 도메인

> 진로 / 컨셉 진단 — 학생이 단계적으로 진행 (progress) → 완료. 강사가 검토하여 approve. 중·고등
> 공용 (FE 의 high-student/concept, middle-student/concept 양쪽 사용).

## 1. 책임

- 학생 본인 진단 진행 (PUT progress)
- 학생 진단 완료 표시 (POST complete)
- 강사 (본인 수업 학생) 진단 조회 + approve

## 2. 핵심 엔티티

| 엔티티 | 핵심 |
| --- | --- |
| `ConceptDiagnosis` | id / studentAccountId (uk, 1:1) / progressJson / status (DRAFT / COMPLETE / APPROVED) / approvedByAccountId / approvedAt |
| `ConceptStatus` | DRAFT / COMPLETE / APPROVED |

## 3. 패키지 구성

```
concept/
├── controller/
│   ├── StudentConceptController     # /api/v1/students/me/concept-diagnosis
│   └── TeacherConceptController     # /api/v1/teachers/me/...
├── service/                         # ConceptDiagnosisService
├── domain/
├── dto/{request,response}/
├── enums/                           # ConceptErrorCode (CON-)
├── exception/
└── handler/
```

## 4. API endpoint

### STUDENT (3)

| Method | Path | 책임 |
| --- | --- | --- |
| GET | `/api/v1/students/me/concept-diagnosis` | 본인 진단 |
| PUT | `/api/v1/students/me/concept-diagnosis/progress` | progress 갱신 (free-form JSON) |
| POST | `/api/v1/students/me/concept-diagnosis/complete` | 완료 (status COMPLETE) |

### TEACHER (2)

| Method | Path | 책임 |
| --- | --- | --- |
| GET | `/api/v1/teachers/me/students/{studentId}/concept-diagnosis` | 학생 진단 조회 |
| POST | `/api/v1/teachers/me/concept-diagnoses/{id}/approve` | approve (DRAFT/COMPLETE → APPROVED) |

## 5. Migration

| V | 내용 |
| --- | --- |
| V25 | `concept_diagnosis` (uk student_account_id + progress_json + status enum + audit) |

## 6. 핵심 흐름

- progress 는 free-form JSON — front-end 의 step 별 응답 저장. 서버는 schema 검증 없이 그대로 저장.
- complete → 학생이 의도적으로 lock. 그 후 강사가 approve.
- approved 후에는 학생이 변경 불가.

## 7. 의존

- **의존**: course (강사 학생 매핑)

## 서비스 로직 (`concept/service/`)

1 service.

| 클래스 | Tx | 책임 |
| --- | --- | --- |
| `ConceptDiagnosisService` | `@Transactional` (write) / `readOnly` (조회) | 학생 진단 진행 + 강사 approve |

### 핵심 메서드

```java
public ConceptDiagnosis getOrCreate(Long studentAccountId)            // findByStudentAccountId or 새 DRAFT INSERT
public ConceptDiagnosis updateProgress(Long studentAccountId, Integer currentQuestion, String progressJson)
                                                          // status=DRAFT 만 허용
public ConceptDiagnosis complete(Long studentAccountId, String typeCode, String typeName, String description)
                                                          // status DRAFT → COMPLETE (잠금)
public ConceptDiagnosis getByStudentForTeacher(Long teacherAccountId, Long studentAccountId)
                                                          // 강사-학생 매핑 검증
public ConceptDiagnosis approveByTeacher(Long teacherAccountId, Long diagnosisId)
                                                          // DRAFT/COMPLETE → APPROVED
```

progressJson 은 free-form (서버 schema 검증 없음).

## cross-link

- [[course]]
