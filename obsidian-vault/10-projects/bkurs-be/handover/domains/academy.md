---
title: "도메인 — academy (학원 / 가입·승인 / 강사 invite / 이관)"
kind: handover
status: stable
created_at: 2026-05-21T23:42:00+09:00
tags: [bkurs-be, handover, domain, academy]
related:
  - "[[../../bkurs-be]]"
  - "[[../catalog]]"
  - "[[../decisions/decision-academy-connect-membership-flow]]"
  - "[[../decisions/decision-academy-management-owner-policy]]"
  - "[[../decisions/decision-teacher-invite-assignment]]"
  - "[[../decisions/decision-owner-transfer-two-party]]"
home_hub: "[[../../bkurs-be]]"
---

# academy 도메인

> "**1인 1학원**" + 학생/강사가 학원에 가입하는 모든 경로 + 강사 invite 7일 1회용 토큰 + 학원장
> 이관 양측 confirm. 본 도메인의 `AcademyMembershipResolver` 는 announcement / dashboard / search
> 의 권한 검증 base.

## 1. 책임

- 학원 entity 1개 = 1 OWNER (`uk_academy_owner_account_id`)
- 학원 가입 코드 random 32-char (재시도 + saveAndFlush)
- 학생 가입: 코드 입력 → `StudentAcademyMembership` PENDING → 학원장 approve → ACTIVE
- 강사 가입: 학원장이 invite 발급 (32-byte base64url, 7일, 1회 consume) → 강사 가입 → `TeacherAcademyAssignment` ACTIVE
- 학원장 이관 양측 confirm — `AcademyOwnerTransfer` (PENDING / CONFIRMED / REJECTED / CANCELLED)
- 학원 정보 4 필드 (phone / address / logoUrl / website) PATCH

## 2. 핵심 엔티티

| 엔티티 | 핵심 |
| --- | --- |
| `Academy` | id / ownerAccountId (uk) / name / 4 info 필드 / `transferTo(newOwner)` invariant method |
| `AcademyCode` | random 32-char (alphabet) + active flag |
| `StudentAcademyMembership` | studentAccountId / academyId / status (PENDING / ACTIVE / REJECTED) — **이중 상태 트랙** (account.status 와 별도) |
| `MembershipStatus` | enum |
| `TeacherInvite` | 32-byte base64url + uk + 7일 expiry + 1회용 (`consume()`) |
| `TeacherAcademyAssignment` | teacherAccountId / academyId / status (ACTIVE / REMOVED) |
| `TeacherAssignmentStatus` | enum |
| `AcademyOwnerTransfer` | fromOwnerId / toAccountId / status + actor invariant |
| `OwnerTransferStatus` | enum |

## 3. 패키지 구성

```
academy/
├── controller/                      # 13개 controller
│   ├── AcademyController            # /api/v1/academies — OWNER 생성 / /me / /me PATCH
│   ├── AcademyCodeController        # /api/v1/academies/me/codes — 코드 발급 + 비활성화
│   ├── ConnectController            # /api/v1/auth/connect — 학생 학원 코드 입력
│   ├── MembershipController         # /api/v1/academies/memberships/{id}/approve|reject
│   ├── MyAcademyController          # /api/v1/academies/my/academy — 학생/강사 본인 학원
│   ├── OwnerMembershipQueueController # /api/v1/academies/me/memberships/pending
│   ├── OwnerStudentListController   # /api/v1/academies/me/students
│   ├── OwnerTransferFromController  # /api/v1/academies/me/owner-transfers (발신 측)
│   ├── OwnerTransferToController    # /api/v1/owner-transfers/incoming (수신 측)
│   ├── StudentMembershipController  # /api/v1/students/me/membership (PR #151)
│   ├── TeacherAssignmentController  # /api/v1/teachers/me/academy + /me/teachers
│   ├── TeacherConnectController     # /api/v1/auth/teacher-connect — 강사 invite token 으로 가입
│   └── TeacherInviteController      # /api/v1/academies/me/teacher-invites
├── service/                         # AcademyConnectService / MembershipService / AcademyManagementService / AcademyCodeService / TeacherInviteService / TeacherAssignmentService / OwnerTransferService
├── domain/                          # entity + Repository (9개)
├── dto/{request,response}/
├── enums/                           # AcademyErrorCode (ACA-)
├── exception/                       # AcademyNotFoundException / OwnerOnlyException ...
└── handler/                         # AcademyExceptionHandler (7 매핑)
```

## 4. 주요 API endpoint

### 학원 본체

| Method | Path | role | 책임 |
| --- | --- | --- | --- |
| POST | `/api/v1/academies` | OWNER | 학원 생성 (1인 1학원 검증) |
| GET | `/api/v1/academies/me` | OWNER | 본인 학원 |
| PATCH | `/api/v1/academies/me` | OWNER | 학원 정보 (4 필드) |

### 코드 / 가입

| Method | Path | role | 책임 |
| --- | --- | --- | --- |
| POST | `/api/v1/academies/me/codes` | OWNER | 코드 발급 |
| GET | `/api/v1/academies/me/codes` | OWNER | 코드 목록 |
| POST | `/api/v1/academies/me/codes/{id}/deactivate` | OWNER | 코드 비활성 |
| POST | `/api/v1/auth/connect` | STUDENT_* | 학생 코드 입력 → membership PENDING |
| POST | `/api/v1/auth/teacher-connect` | TEACHER | 강사 invite token 입력 |

### 승인 / 큐

| Method | Path | role | 책임 |
| --- | --- | --- | --- |
| GET | `/api/v1/academies/me/memberships/pending` | OWNER | 대기 큐 |
| POST | `/api/v1/academies/memberships/{id}/approve` | OWNER | 승인 |
| POST | `/api/v1/academies/memberships/{id}/reject` | OWNER | 거절 |
| GET | `/api/v1/academies/me/students` | OWNER | 학생 목록 |
| GET | `/api/v1/students/me/membership` | STUDENT_* | 본인 마지막 membership (academy join, PR #151) |

### 강사

| Method | Path | role | 책임 |
| --- | --- | --- | --- |
| POST | `/api/v1/academies/me/teacher-invites` | OWNER | invite 발급 |
| GET | `/api/v1/academies/me/teacher-invites` | OWNER | invite 목록 |
| GET | `/api/v1/academies/me/teachers` | OWNER | 강사 목록 |
| POST | `/api/v1/academies/me/teachers/{assignmentId}/remove` | OWNER | 강사 해제 |
| GET | `/api/v1/teachers/me/academy` | TEACHER | 본인 배정 학원 |

### 학원장 이관

| Method | Path | role | 책임 |
| --- | --- | --- | --- |
| POST | `/api/v1/academies/me/owner-transfers` | OWNER (from) | 이관 신청 |
| GET | `/api/v1/academies/me/owner-transfers` | OWNER (from) | 진행 중 목록 |
| POST | `/api/v1/academies/me/owner-transfers/{id}/cancel` | OWNER (from) | 신청 취소 |
| GET | `/api/v1/owner-transfers/incoming` | (수신 후보) | 본인 수신 목록 |
| POST | `/api/v1/owner-transfers/{id}/confirm` | (수신 후보) | 수락 — 학원 owner 변경 |
| POST | `/api/v1/owner-transfers/{id}/reject` | (수신 후보) | 거절 |

## 5. Migration

| V | 내용 |
| --- | --- |
| V4 | `academy` + `academy_code` (uk) + `student_academy_membership` (이중 상태) |
| V5 | `teacher_invite` (uk token, 7일) + `teacher_academy_assignment` (ACTIVE/REMOVED) |
| V7 | `ALTER academy` ADD phone/address/logo_url/website (모두 nullable) |
| V10 | `academy_owner_transfer` (PENDING/CONFIRMED/REJECTED/CANCELLED + actor invariant) |

## 6. 핵심 invariant

### 1인 1학원 (`AcademyManagementService.createForOwner`)

DB: `uk_academy_owner_account_id` — 두 학원 같은 owner 금지.
service: 생성 전 `existsByOwnerAccountId(ownerId)` 체크 + 충돌 시 `OwnerAlreadyHasAcademyException`.

### 학원장 이관 양측 confirm (`OwnerTransferService.confirm`)

```
1. transfer.status = PENDING 이어야
2. confirmer = transfer.toAccountId 이어야
3. toAccount 가 이미 학원 owner 면 reject
4. transferStatus = CONFIRMED, academy.transferTo(newOwner) — 한 트랜잭션
```

actor invariant: 본인 거 외에는 confirm 못 함 (path 와 principal 일치).

### 강사 invite 1회용 (`TeacherInviteService.consume`)

```
invite.expiresAt > now AND invite.consumedAt == null
→ assignment INSERT (ACTIVE)
→ invite.consume()  ─ consumedAt = now
```

같은 token 두 번 사용 시 `InvalidInviteTokenException`.

### code random 충돌 (B3A 회수)

`AcademyCodeService` 가 random 32-char 생성 → `saveAndFlush` 로 즉시 unique 검증 → 충돌 시 다시
생성 (최대 N회). `save` 만 쓰면 트랜잭션 끝 시점에 충돌 → 재시도 어려움.

## 7. AcademyMembershipResolver — 의존받음의 핵심

위치는 `announcement/service/AcademyMembershipResolver.java` (코드 의존상 announcement 모듈에
있지만, **본 도메인 데이터 기반**). role 별 분기:

```
OWNER          → 본인이 owner_account_id 인 academy
TEACHER        → ACTIVE teacher_academy_assignment 의 academy
STUDENT_HIGH/M → ACTIVE student_academy_membership 의 academy
MASTER         → empty (별도 endpoint)
```

announcement / dashboard / search 가 의존.

## 8. 의존

- **의존**: auth (Account / Role)
- **의존받음**: announcement, student, plan, course, dashboard, search, stats, record, 모든 학습 도메인
- **cross-domain**: 학원 내부 학생 / 강사 권한 검증의 진입점

## 9. 회수

| 회수 | PR |
| --- | --- |
| `save` vs `saveAndFlush` 충돌 검출 시점 (B3A code 충돌) | saveAndFlush 로 즉시 unique violation |
| 학원 정보 4 필드 추가 후 GlobalValidationExceptionHandler 누락 → ERROR 401 (B4A) | handler Order 정리 |

## 서비스 로직 (`academy/service/`)

**7 service** — 가장 service 가 많은 도메인. 가입 / 코드 / 학원장 관리 / 강사 invite / 이관 별로 분리.

| 클래스 | Tx | 책임 |
| --- | --- | --- |
| `AcademyManagementService` | `@Transactional` | 학원 entity 1인 1학원 invariant CRUD |
| `AcademyCodeService` | `@Transactional` | 32-char random code 발급 + saveAndFlush 충돌 재시도 + deactivate |
| `AcademyConnectService` | `@Transactional` | 학생 코드 입력 → `StudentAcademyMembership` PENDING 생성 |
| `MembershipService` | `@Transactional` | 학원장 approve / reject (ownership 2계층 검증) |
| `TeacherInviteService` | `@Transactional` | invite 발급 (32-byte base64url + 7일) + 자동 이메일 + consume |
| `TeacherAssignmentService` | mostly `readOnly` | 강사 본인 active 배정 / 학원장의 강사 목록 / remove |
| `OwnerTransferService` | `@Transactional` | 학원장 이관 — initiate / confirm / reject / cancel + 양측 actor invariant |

### 핵심 메서드

```java
// AcademyManagementService
public Academy create(Long ownerAccountId, String name)            // existsByOwnerAccountId → 충돌 시 OwnerAlreadyHasAcademyException
public Academy getMine(Long ownerAccountId)
public Academy updateInfoMine(Long ownerAccountId, String phone, String address, String logoUrl, String website)  // 4 nullable

// AcademyCodeService
public AcademyCode issue(Long ownerAccountId)                       // random 32-char + saveAndFlush → uk 위반 catch → 재시도
public AcademyCode deactivate(Long ownerAccountId, Long codeId)

// AcademyConnectService
public ConnectResult connect(Long accountId, String code)           // code 조회 → academy 결정 → membership PENDING INSERT + account.status = PENDING_APPROVE

// MembershipService
public StudentAcademyMembership approve(Long membershipId, Long ownerAccountId)  // ownership 검증 + status ACTIVE + account.status ACTIVE
public StudentAcademyMembership reject(Long membershipId, Long ownerAccountId, String reason)

// TeacherInviteService
public TeacherInvite issueAndNotify(Long ownerAccountId, String inviteEmail)  // INSERT + AsyncNotificationDispatcher.dispatch (email)
public TeacherAcademyAssignment connect(Long accountId, String token)         // invite.consume() invariant + assignment ACTIVE INSERT

// OwnerTransferService
public AcademyOwnerTransfer initiate(Long fromOwnerAccountId, String toEmail)         // 본인 학원 검증 + toAccount lookup + PENDING INSERT
public AcademyOwnerTransfer confirm(Long toAccountId, Long transferId)                // status PENDING + actor=to 검증 + academy.transferTo(...) + status CONFIRMED — 한 TX
public AcademyOwnerTransfer reject(Long toAccountId, Long transferId)
public AcademyOwnerTransfer cancel(Long fromAccountId, Long transferId)
```

핵심 invariant: code random 충돌 시 `save` 가 아닌 `saveAndFlush` 로 즉시 unique violation 받기 (B3A 회수). 이관은 양측 confirm 필수 + 1인 1학원 유지.

## cross-link

- [[auth]] — 학생/강사 가입 status 흐름
- [[announcement]] — `AcademyMembershipResolver` 위치 + 학원 공지
- [[student]] — `StudentProfile` 의 academy 의존
- [[plan]] — 학원 plan 구독 / 결제
- [[../decisions/decision-academy-connect-membership-flow]]
- [[../decisions/decision-academy-management-owner-policy]]
- [[../decisions/decision-academy-info-fields]]
- [[../decisions/decision-teacher-invite-assignment]]
- [[../decisions/decision-owner-transfer-two-party]]
