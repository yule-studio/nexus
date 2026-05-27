---
title: "B3C — 학원 공지 + read 추적 작업 로그"
kind: task-log
project: bkurs-be
session_id: bkurs-be-b3-academy
created_at: 2026-05-20T08:00:00+09:00
agent: engineering-agent/tech-lead
status: in-progress
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-announcement-read-tracking]]"
  - "[[task-log-b3b-teacher-invite]]"
tags: [task-log, announcement, read-tracking, b3c]
---

# 작업

GitHub issue [#13](https://github.com/seum-running/bkurs-be/issues/13) — 학원 공지 + read 추적.

# 결정 요약 (인수인계용)

| 결정 | 내용 | 상세 |
|---|---|---|
| 데이터 모델 | 별도 `announcement_read` 테이블, (announcement_id, account_id) UNIQUE | [[../decisions/decision-announcement-read-tracking]] |
| 작성 권한 | OWNER 만 (B3C 기준) | TEACHER 작성은 B4+ 강사 권한 매트릭스 |
| read 추적 | 명시적 POST /read 호출 시만 upsert | GET 부수효과 회피 (REST 원칙) |
| 본인 학원 ID | AcademyMembershipResolver — role 별 통합 | OWNER/TEACHER/STUDENT 분기 |
| URL prefix | /me (학원장 관점) vs /my (학생·강사 본인 학원 관점) | 의미 구분 |

# 진행

## 2026-05-20

### 1. branch / issue
- B3B 머지 대기 중 stacked PR — base=feature/issue-11-b3b-teacher-invite
- issue #13 생성 + decision 노트 (4 대안 × 2 — 데이터 모델 + 작성 권한)

### 2. 코드 (commit 단위)
- C1: Announcement + AnnouncementRead entity + V6 + 2 Repository + update/belongsTo invariant 도메인 메서드
- C2: 2 예외 + AnnouncementService (OWNER CRUD + ownership) + AnnouncementReadService (listForMember N+1 회피 + markRead upsert) + AcademyMembershipResolver (role 별 통합)
- C3: 4 DTO + 2 controller (OwnerController /me OWNER + MemberController /my role 무관) + AnnouncementExceptionHandler
- C4: AnnouncementApiTest 10 회귀 + 3 기존 테스트 cleanup 보강 (announcement_*) + CHANGELOG B3C
- C5: CHANGELOG fixup (직전 commit 누락분, docs only)

### 3. PR
- PR [#14](https://github.com/seum-running/bkurs-be/pull/14) (feature/issue-13 → feature/issue-11-b3b-teacher-invite, stacked)
- 회귀 **54/54 GREEN**

# 인수인계 (Handover) — 왜 이렇게 만들었나

| 구현 | 선택 이유 | 대안 / 후속 |
|---|---|---|
| 별도 `announcement_read` 테이블 | announcement.read_account_ids JSON 안티패턴 회피. 정규화 + (announcement, account) 인덱스로 양방향 query 효율. | read_count 컬럼 + 별도 N rows 는 hot path 발생 시. B3C 단순 정규화 우선. |
| (announcement_id, account_id) UNIQUE | upsert key — 같은 사용자가 /read 여러 번 호출해도 1 row, 멱등. | UNIQUE 없이 application 검증은 race condition 위험. DB 레벨 강제. |
| announcement → read CASCADE | 공지 삭제 시 read row 자동 정리 — orphan 데이터 방지. application 정리 코드 불필요. | manual 삭제 패턴은 누락 위험. FK CASCADE 가 정답. |
| OWNER 만 작성 (B3C) | MVP 정책, 가장 안전한 default. 강사 권한은 매트릭스 별도 결정 필요. | B4+ 의 강사 권한 매트릭스 PR 후 풀 가능. entity author_id 는 미리 academy 외 role 허용 (스키마 유연성). |
| GET 부수효과 X (명시적 /read 호출) | REST 원칙 — GET 은 idempotent + safe. 자동 read 추적은 캐시 안전성 깨짐. | 클라이언트가 viewport 진입 시 자동 호출 (intersection observer) 패턴 권장. |
| `AcademyMembershipResolver` 신규 service | role 별 본인 학원 id 해결 로직 분산 시 복제. 단일 진입점 + switch (role) 패턴. | B4+ 의 학원 정보 / 수업 / 일정 endpoint 도 본 resolver 재사용. |
| 학원 미가입 → 빈 목록 (404 X) | 정보 노출 최소화. 404 면 "학원 가입 안 되어있음" 정보 제공 = 권한 정찰 가능. | 학생 PENDING_CONNECT / 강사 unassigned 모두 빈 목록 동일. |
| listForMember 의 IN query (N+1 회피) | 공지 N + read 1 query (announcement_id IN list) → in-memory join. JPA fetch join 보다 단순. | 공지 10k 까지 늘면 page 별 IN. 가정 100/학원 기준 안전. |
| /me vs /my URL prefix | /me = 학원장 관점 (소유 학원의 공지 CRUD), /my = 학생·강사 본인 학원 관점 (조회+read). | 같은 prefix 면 학원장 / 학생 endpoint 혼재 → 라우팅 모호. |
| `AnnouncementMemberController` role 무관 (OWNER 포함) | OWNER 도 본인 학원 공지 GET 가능 (편의). resolver 가 OWNER → owner 학원 ID 반환. | 별도 분기 안 함 — single resolver, simpler. |

# 다음 단계 (예정)

- B2A → B2B → B2C → B3A → B3B → B3C 순으로 사용자 manual merge → CI 자동 실행
- B4 시작 — 플랜 + 결제 + 학원 이관 + 학원 정보 (전화/주소/로고)
- B5 — 수업 / 일정 본격 도메인

# 회수 / pitfall (누적)

- V6 FK 누적 — 6 FK 추가로 기존 5 테스트 cleanup 의 announcement_*Repository 추가 필요. CASCADE 가 일부 의존 자동 처리하나 명시적 정리가 안전.
- AcademyMembershipResolver 같은 cross-domain resolver — 도메인 (announcement) 이 다른 도메인 (academy/student/teacher) 의 상태를 의존. 향후 ACL / 권한 정책이 복잡해지면 별도 패키지 (`com.bkurs.access`) 로 분리 후보.

# 참고

- [[../decisions/decision-announcement-read-tracking]]
- [[task-log-b3b-teacher-invite]]
