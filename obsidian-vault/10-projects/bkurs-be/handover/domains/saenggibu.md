---
title: "도메인 — saenggibu (중등 생기부 + 학기 잠금)"
kind: handover
status: stable
created_at: 2026-05-22T00:01:00+09:00
tags: [bkurs-be, handover, domain, saenggibu]
related:
  - "[[../../bkurs-be]]"
  - "[[record]]"
  - "[[../decisions/decision-saenggibu-semester-lock]]"
home_hub: "[[../../bkurs-be]]"
---

# saenggibu 도메인

> 중등 생기부. 학기 단위 free-form 작성 — **학원장이 학기 lock 을 걸면 그 학기는 readonly**.
> 고등 record 와 분리 (record 는 8 카테고리 + bridge, saenggibu 는 학기 free-form).

## 1. 책임

- 학생 본인 생기부 학기별 free-form 작성 / 조회
- 학원장이 학원 학기별 lock 발행 — 잠긴 학기는 학생이 PATCH 불가
- 강사가 학원 내 학생 생기부 조회

## 2. 핵심 엔티티

| 엔티티 | 핵심 |
| --- | --- |
| `MiddleSaenggibu` | id / studentAccountId / academyId / semester (year+sem string) / content / createdAt / updatedAt |
| `MiddleSaenggibuLock` | composite key (academy_id, semester) + lockedAt / lockedByAccountId — `MiddleSaenggibuLockId` 가 ID 클래스 |

## 3. 패키지 구성

```
saenggibu/
├── controller/
│   ├── OwnerSaenggibuController     # /api/v1/academies/{academyId}/saenggibu/lock — lock 발행/조회
│   ├── StudentSaenggibuController   # /api/v1/students/me/saenggibu — POST/GET
│   └── TeacherSaenggibuController   # /api/v1/teachers/academies/{academyId}/saenggibu — GET
├── service/
├── domain/                          # MiddleSaenggibu, MiddleSaenggibuLock, MiddleSaenggibuLockId (composite PK)
├── dto/{request,response}/
├── enums/                           # SaenggibuErrorCode (SGB-)
├── exception/
└── handler/
```

## 4. API endpoint

### OWNER (2)

| Method | Path | 책임 |
| --- | --- | --- |
| POST | `/api/v1/academies/{academyId}/saenggibu/lock` | 학기 lock 발행 |
| GET | `/api/v1/academies/{academyId}/saenggibu/lock` | 학기 lock 목록 |

### STUDENT (2)

| Method | Path | 책임 |
| --- | --- | --- |
| POST | `/api/v1/students/me/saenggibu` | 생기부 upsert (잠긴 학기 X) |
| GET | `/api/v1/students/me/saenggibu` | 본인 생기부 목록 |

### TEACHER (1)

| Method | Path | 책임 |
| --- | --- | --- |
| GET | `/api/v1/teachers/academies/{academyId}/saenggibu` | 학원 학생 생기부 조회 |

## 5. Migration

| V | 내용 |
| --- | --- |
| V27 | `middle_saenggibu` + `middle_saenggibu_lock` (composite PK) |

## 6. 핵심 invariant

### 학기 lock 검증

```
StudentSaenggibu.upsert(studentId, semester, content)
  ▼
academyId = membership.findActiveByStudentId(studentId).academyId
if (lockRepo.existsByIdAcademyIdAndIdSemester(academyId, semester)) {
    throw new SaenggibuLockedException(semester);  // 403
}
saenggibuRepo.findByStudentIdAndSemester(...)
   .ifPresentOrElse(e -> e.update(content), () -> save(new ...));
```

lock 발행 후 더 이상 update 불가. 학원장이 학기 마감 정책.

### composite PK

`MiddleSaenggibuLockId` = `(academyId, semester)`. JPA `@IdClass` 또는 `@EmbeddedId` — 어느 방식을
썼는지는 코드 참조.

## 7. 의존

- **의존**: auth, academy (membership)
- **연관**: suhaeng (생기부 작성 시 suhaeng 결과 활용 가능 — 자동 bridge 는 없음, 학생이 manually 옮김)

## 8. record 와 분리

| 도메인 | 학교급 | 구조 |
| --- | --- | --- |
| `record` | 고등 | 8 카테고리 entry + 4 도메인 자동 bridge |
| `saenggibu` | 중등 | 학기 free-form text + 학기 lock |

## 서비스 로직 (`saenggibu/service/`)

1 service.

| 클래스 | Tx | 책임 |
| --- | --- | --- |
| `SaenggibuService` | `@Transactional` (write) / `readOnly` (조회) | 학생 upsert (lock 검증) + 학원장 lock 발행 |

### 핵심 메서드

```java
public MiddleSaenggibu upsertMine(Long studentAccountId, Long academyId, Integer grade, Integer semester, String content)
                                                          // 1. membership ACTIVE 검증
                                                          // 2. lockRepo.existsById(LockId(academyId, grade, semester)) → throw 면 SaenggibuLockedException (403)
                                                          // 3. findBySaenggibu → 갱신 또는 INSERT
public List<MiddleSaenggibu> listMineByTerm(Long studentAccountId, Integer grade, Integer semester)

public MiddleSaenggibuLock setLock(Long ownerAccountId, Long academyId, Integer grade, Integer semester)
                                                          // 학원장 본인 학원 검증 + composite PK INSERT
public MiddleSaenggibuLock   getLock(Long academyId, Integer grade, Integer semester)

public List<MiddleSaenggibu> listByAcademyForTeacher(Long teacherAccountId, Long academyId, Integer grade, Integer semester)
                                                          // 강사 본인 학원 배정 검증
```

lock 발행 후 학생 PATCH 불가. 학기 마감 정책.

## cross-link

- [[record]] — 고등 대응
- [[suhaeng]] — 학생 자료
- [[../decisions/decision-saenggibu-semester-lock]]
