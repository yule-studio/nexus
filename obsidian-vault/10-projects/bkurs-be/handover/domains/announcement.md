---
title: "도메인 — announcement (학원 공지 + 읽음 추적)"
kind: handover
status: stable
created_at: 2026-05-21T23:43:00+09:00
tags: [bkurs-be, handover, domain, announcement]
related:
  - "[[../../bkurs-be]]"
  - "[[../decisions/decision-announcement-read-tracking]]"
home_hub: "[[../../bkurs-be]]"
---

# announcement 도메인

> 학원장이 학원 회원 (학생/강사) 에게 공지. 회원 별 읽음 추적 (uk + CASCADE).
> `AcademyMembershipResolver` 가 본 도메인 안에 있고 다른 도메인이 의존하는 특이점.

## 1. 책임

- OWNER 학원 공지 CRUD
- 회원 (학생/강사) 읽음 표시 (한 번만, uk 강제)
- role 무관 `/me` 공지 통합 — `AcademyMembershipResolver` 로 본인 학원 결정

## 2. 핵심 엔티티

| 엔티티 | 핵심 |
| --- | --- |
| `Announcement` | id / academyId / title / body / createdAt / updatedAt |
| `AnnouncementRead` | announcementId / accountId / readAt + uk(announcement_id, account_id) + ON DELETE CASCADE |

## 3. 패키지 구성

```
announcement/
├── controller/
│   ├── AnnouncementOwnerController  # /api/v1/academies/me/announcements — OWNER CRUD
│   └── AnnouncementMemberController # /api/v1/academies/my/announcements — 학생/강사 공통
├── service/
│   ├── AnnouncementService           # OWNER CRUD
│   ├── AnnouncementReadService       # 회원 read mark
│   └── AcademyMembershipResolver     # role → academyId 통합 (다른 도메인도 사용)
├── domain/
├── dto/{request,response}/
├── enums/      # AnnouncementErrorCode (ANN-)
├── exception/
└── handler/    # AnnouncementExceptionHandler
```

## 4. API endpoint

### OWNER (CRUD)

| Method | Path | 책임 |
| --- | --- | --- |
| POST | `/api/v1/academies/me/announcements` | 공지 생성 |
| GET | `/api/v1/academies/me/announcements` | 본인 학원 공지 목록 |
| PATCH | `/api/v1/academies/me/announcements/{id}` | 수정 |
| DELETE | `/api/v1/academies/me/announcements/{id}` | 삭제 (AnnouncementRead CASCADE 자동 정리) |

### 학생/강사 (`/my/`)

| Method | Path | 책임 |
| --- | --- | --- |
| GET | `/api/v1/academies/my/announcements` | 본인 학원 공지 목록 (read 여부 포함) |
| POST | `/api/v1/academies/my/announcements/{id}/read` | 읽음 표시 (upsert idempotent) |

## 5. Migration

| V | 내용 |
| --- | --- |
| V6 | `announcement` + `announcement_read` (uk + FK CASCADE) |

## 6. 핵심 코드 흐름

### read mark (upsert)

```
AnnouncementReadService.markRead(announcementId, accountId)
  ▼
existsByAnnouncementIdAndAccountId(id, aid) ?
  ├── 있음 → no-op (idempotent)
  └── 없음 → INSERT
```

uk 충돌 시 idempotent silent — 동시 클릭으로 두 번 들어와도 한 row.

### N+1 회피

`/my/announcements` 응답에 read 여부 포함 — `findAllWithReadByAccount(accountId)` 한 쿼리로 join.
read 표시는 List<long> 으로 따로 조회 후 매핑.

### CASCADE

announcement DELETE 시 read row 도 자동 삭제 — `ON DELETE CASCADE`. 별도 service 로직 없음.

## 7. AcademyMembershipResolver

본 도메인 안에 있지만 cross-domain 핵심. `account.role` 에 따라:

```
OWNER          → academy.findByOwnerAccountId(...)
TEACHER        → teacherAcademyAssignment.findByTeacherIdAndStatus(id, ACTIVE)
STUDENT_*      → studentAcademyMembership.findByStudentIdAndStatus(id, ACTIVE)
MASTER         → Optional.empty()
```

→ caller 가 "본인 학원" 단위 조회 / 권한 검증 할 때 본 메서드만 호출.

사용처: announcement (자기 자신), dashboard.OwnerDashboardService, search (도메인 간), stats.

## 8. 의존

- **의존**: auth (Account / Role), academy (Academy / Membership / Assignment Repository)
- **의존받음**: dashboard, search, stats

## 서비스 로직 (`announcement/service/`)

3 service. `AcademyMembershipResolver` 는 본 도메인 안에 있지만 cross-domain 핵심.

| 클래스 | Tx | 책임 |
| --- | --- | --- |
| `AnnouncementService` | `@Transactional` | OWNER 공지 CRUD |
| `AnnouncementReadService` | `@Transactional` | 회원 read mark (idempotent upsert) + listForMember (N+1 회피) |
| `AcademyMembershipResolver` | `@Transactional(readOnly)` | role → academyId 통합 매핑 (dashboard / search / stats 가 의존) |

### 핵심 메서드

```java
// AnnouncementService
public Announcement create(Long ownerAccountId, String title, String body)   // 본인 학원 검증 + INSERT
public List<Announcement> listByOwner(Long ownerAccountId)
public Announcement update(Long ownerAccountId, Long id, String title, String body)
public void delete(Long ownerAccountId, Long id)                              // FK CASCADE 로 AnnouncementRead 자동 정리

// AnnouncementReadService
public List<AnnouncementWithRead> listForMember(Long academyId, Long accountId)  // join 1 쿼리로 read 여부 동봉
public AnnouncementRead markRead(Long accountId, Long academyId, Long announcementId)  // existsBy 체크 → idempotent INSERT

// AcademyMembershipResolver
public Optional<Long> resolveAcademyId(Long accountId)  // role 별 분기:
                                                       //   OWNER → academy.findByOwnerAccountId
                                                       //   TEACHER → teacherAssignment ACTIVE
                                                       //   STUDENT_* → membership ACTIVE
                                                       //   MASTER → empty
```

## cross-link

- [[academy]] — Resolver 의 데이터 소스
- [[../decisions/decision-announcement-read-tracking]]
- [[../common]] — envelope / handler
