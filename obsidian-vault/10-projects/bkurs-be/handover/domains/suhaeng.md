---
title: "도메인 — suhaeng (중등 수행/면담 라운드)"
kind: handover
status: stable
created_at: 2026-05-22T00:00:00+09:00
tags: [bkurs-be, handover, domain, suhaeng]
related:
  - "[[../../bkurs-be]]"
  - "[[../decisions/decision-suhaeng-round-table]]"
home_hub: "[[../../bkurs-be]]"
---

# suhaeng 도메인

> 중등 학생 수행/면담 라운드. 한 학생 N round — 강사 발행 → 학생 제출 / 재제출 → 강사 feedback
> → 강사 finalize. 학원장이 학원 내 suhaeng 발행 가능.

## 1. 책임

- OWNER: 학원 내 suhaeng 발행
- STUDENT: 본인 round 목록 + 제출 + 재제출
- TEACHER: 학원 내 suhaeng 조회 + round feedback + finalize

## 2. 핵심 엔티티

| 엔티티 | 핵심 |
| --- | --- |
| `Suhaeng` | id / academyId / title / round 정의 등 (수업/면담 종류) |
| `SuhaengRound` | id / suhaengId / studentAccountId / submission / feedback / finalizedAt — 학생당 N round |

## 3. 패키지 구성

```
suhaeng/
├── controller/
│   ├── OwnerSuhaengController       # /api/v1/academies/{academyId}/suhaeng (POST)
│   ├── StudentSuhaengController     # /api/v1/students/me/suhaeng (GET / submit / resubmit)
│   └── TeacherSuhaengController     # /api/v1/teachers/academies/{academyId}/suhaeng (GET / feedback / finalize)
├── service/
├── domain/
├── dto/{request,response}/
├── enums/                           # SuhaengErrorCode (SHN-)
├── exception/
└── handler/
```

## 4. API endpoint

### OWNER (1)

| Method | Path | 책임 |
| --- | --- | --- |
| POST | `/api/v1/academies/{academyId}/suhaeng` | suhaeng 발행 (학원 내) |

### STUDENT (4)

| Method | Path | 책임 |
| --- | --- | --- |
| GET | `/api/v1/students/me/suhaeng` | 내 suhaeng 목록 |
| GET | `/api/v1/students/me/suhaeng/{id}` | 상세 |
| POST | `/api/v1/students/me/suhaeng/{id}/submit` | 제출 |
| POST | `/api/v1/students/me/suhaeng/{id}/resubmit` | 재제출 |

### TEACHER (3)

| Method | Path | 책임 |
| --- | --- | --- |
| GET | `/api/v1/teachers/academies/{academyId}/suhaeng` | 학원 suhaeng 조회 |
| POST | `/api/v1/teachers/suhaeng-rounds/{roundId}/feedback` | round feedback |
| POST | `/api/v1/teachers/suhaeng-rounds/{roundId}/finalize` | 마무리 (잠금) |

## 5. Migration

| V | 내용 |
| --- | --- |
| V26 | `suhaeng` + `suhaeng_round` |

## 6. 의존

- **의존**: auth, academy
- **연관**: middle-saenggibu (생기부 작성 시 suhaeng 결과 참조 가능)

## 서비스 로직 (`suhaeng/service/`)

1 service.

| 클래스 | Tx | 책임 |
| --- | --- | --- |
| `SuhaengService` | `@Transactional` (write) / `readOnly` (조회) | OWNER 발행 + 학생 round 제출 / 재제출 + 강사 feedback / finalize |

### 핵심 메서드

```java
public Suhaeng createByOwner(Long ownerAccountId, Long academyId, CreateSuhaengRequest req)  // 학원장 본인 학원 검증
public List<Suhaeng> listMine(Long studentAccountId)                  // membership ACTIVE 학원의 suhaeng
public Suhaeng       getForStudent(Long studentAccountId, Long suhaengId)

public SuhaengRound submitFirst(Long studentAccountId, Long suhaengId, String answer)
                                                          // round 1 시작 — 이미 round 있으면 throw
public SuhaengRound resubmit(Long studentAccountId, Long suhaengId, String answer)
                                                          // 마지막 round 가 finalized 이면 새 round, 아니면 기존 갱신
public SuhaengRound recordFeedback(Long teacherAccountId, Long roundId, String feedback)
                                                          // 강사 권한 + feedback 작성
public SuhaengRound finalizeRound(Long teacherAccountId, Long roundId, Integer score)
                                                          // 마무리 — 학생이 더 이상 이 round 수정 불가
public List<Suhaeng>      listByAcademyForTeacher(Long teacherAccountId, Long academyId)
public List<SuhaengRound> roundsOf(Long suhaengId)
```

## cross-link

- [[saenggibu]]
- [[../decisions/decision-suhaeng-round-table]]
