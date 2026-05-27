---
title: "도메인 — student (학생 프로필)"
kind: handover
status: stable
created_at: 2026-05-21T23:44:00+09:00
tags: [bkurs-be, handover, domain, student]
related:
  - "[[../../bkurs-be]]"
  - "[[../decisions/decision-student-profile-school-grade]]"
home_hub: "[[../../bkurs-be]]"
---

# student 도메인

> 학생 본인의 프로필 (school / grade / birthDate / parentPhone). account 와 1:1. 학원장이 학원 내
> 학생 프로필을 조회할 때도 본 도메인을 통과.

## 1. 책임

- 학생 본인 프로필 CRUD (`/api/v1/students/me/profile`)
- 학원장이 학원 내 학생 프로필 조회 (`/api/v1/academies/me/students/{id}/profile`)
- grade 범위 1~6 (PR #150 에서 1~3 → 1~6 확장)

## 2. 핵심 엔티티

| 엔티티 | 핵심 |
| --- | --- |
| `StudentProfile` | id / accountId (uk, 1:1) / schoolName / grade (INT 1~6) / birthDate / parentPhone |

> grade 컬럼 type 은 `INT` — Java `Integer` 와 매핑. `TINYINT` 로 두면 JPA validate 실패 (B5 회수).

## 3. 패키지 구성

```
student/
├── controller/
│   ├── StudentProfileController        # /api/v1/students/me/profile (POST/GET/PATCH)
│   └── OwnerStudentProfileController   # /api/v1/academies/me/students/{id}/profile (GET)
├── service/                            # StudentProfileService — upsert / patch / get + ownership 2계층
├── domain/                             # StudentProfile + Repository
├── dto/{request,response}/             # UpsertProfileRequest, PatchProfileRequest, StudentProfileResponse
├── enums/                              # StudentErrorCode (STU-)
├── exception/
└── handler/
```

## 4. API endpoint

| Method | Path | role | 책임 |
| --- | --- | --- | --- |
| POST | `/api/v1/students/me/profile` | STUDENT_* | upsert (create / replace) |
| GET | `/api/v1/students/me/profile` | STUDENT_* | 본인 프로필 |
| PATCH | `/api/v1/students/me/profile` | STUDENT_* | 부분 수정 |
| GET | `/api/v1/academies/me/students/{studentAccountId}/profile` | OWNER | 학원 내 학생 프로필 (ownership 검증) |

## 5. Migration

| V | 내용 |
| --- | --- |
| V11 | `student_profile` (uk account_id, 1:1) — grade range 1~6 (PR #150 entity 변경, 컬럼 변경 없음) |

## 6. 핵심 코드 흐름

### ownership 2계층 (OWNER 조회)

```
OwnerStudentProfileController.get(studentAccountId)
  ▼
1. @PreAuthorize("hasRole('OWNER')")              ─ role 검증
2. academyService.getMine(ownerId)                ─ 본인 학원 결정
3. membership.existsByStudentIdAndAcademyIdAndStatus(studentId, academyId, ACTIVE)
                                                  ─ 학생이 본인 학원 소속인지
4. studentProfileRepository.findByAccountId(...)
```

학생이 학원 회원 아니면 `ForbiddenException` (403).

### upsert (학생 본인)

`POST /api/v1/students/me/profile` — 기존 row 있으면 replace, 없으면 INSERT. `findByAccountId` →
존재하면 모든 필드 갱신 + dirty checking flush.

### grade 범위 확장 (PR #150)

```diff
- @Max(value = 3, message = "...")
+ @Max(value = 6, message = "...")
private Integer grade;
```

DTO `UpsertProfileRequest` / `PatchProfileRequest` 양쪽 + entity 의 validation 도 갱신. 컬럼 type
INT 이라 schema 변경 없음.

## 7. 의존

- **의존**: auth (Account / Role), academy (Membership Repository)
- **의존받음**: course (강사가 학생 프로필 조회), stats, dashboard, admin/students/detail (FE)

## 서비스 로직 (`student/service/`)

1 service — `StudentProfileService`. ownership 2계층 검증을 본 service 가 제공.

| 클래스 | Tx | 책임 |
| --- | --- | --- |
| `StudentProfileService` | `@Transactional` (write), `readOnly` (조회) | 본인 upsert/patch + 학원장 조회 ownership 검증 |

### 핵심 메서드

```java
public StudentProfile upsertMine(Long accountId, Integer grade, String schoolName,
                                  LocalDate birthDate, String parentPhone)
                                                          // findByAccountId → 있으면 모든 필드 갱신, 없으면 INSERT
public StudentProfile patchMine(Long accountId, Integer grade, ...)         // 부분 수정 (nullable 인자만 적용)
public StudentProfile getMine(Long accountId)
public StudentProfile getByOwner(Long ownerAccountId, Long studentAccountId)
                                                          // 1. 학원장 본인 학원 결정
                                                          // 2. membership.existsByStudentIdAndAcademyIdAndStatus ACTIVE
                                                          // 3. 통과 시 학생 프로필 반환, 실패 시 ForbiddenException
public StudentProfile getStudentOnly(Long studentAccountId)  // course.TeacherStudentProfileController 에서 사용 — 강사가 본인 수업 학생 프로필
```

`getStudentOnly` 는 course 도메인이 자체 권한 검증 후 호출하는 helper (강사 수업 매핑 검증은 course service 가).

## cross-link

- [[auth]]
- [[academy]]
- [[course]] — `TeacherStudentProfileController` 가 강사 학생 프로필 조회
- [[../decisions/decision-student-profile-school-grade]]
