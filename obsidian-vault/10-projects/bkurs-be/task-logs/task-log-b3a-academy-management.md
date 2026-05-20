---
title: "B3A — 학원 CRUD + 학원 코드 발급/비활성화 작업 로그"
kind: task-log
project: bkurs-be
session_id: bkurs-be-b3-academy
created_at: 2026-05-20T06:30:00+09:00
agent: engineering-agent/tech-lead
status: in-progress
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-academy-management-owner-policy]]"
  - "[[../decisions/decision-academy-connect-membership-flow]]"
  - "[[task-log-b2c-academy-connect]]"
tags: [task-log, academy, owner, code-management, b3a]
---

# 작업

GitHub issue [#9](https://github.com/seum-running/bkurs-be/issues/9) — 학원 CRUD + 학원 코드 발급/비활성화/목록 + 학원장 큐.

# 결정 요약 (인수인계용)

| 결정 | 내용 | 상세 |
|---|---|---|
| 학원 1:N | 학원장 1명 = 학원 1개 (B3A) | [[../decisions/decision-academy-management-owner-policy]] |
| 코드 정책 | 학원당 다중 active code 허용 | 분기별/강사용/신학기 등 학원 운영 현실 반영 |
| 코드 생성 | 32-char alphabet × 8자 random + 충돌 재시도 | ambiguous 제외 (0/O/I/1), uk_academy_code 충돌 시 최대 5회 |
| 권한 | 클래스 레벨 @PreAuthorize OWNER + service ownership 검증 | role 1차 + ownership 2차 (다른 학원장 차단) |
| PATCH | name 만 | owner 변경 / 학원 이관은 별도 endpoint (B4+) |

# 진행

## 2026-05-20

### 1. branch / issue
- B2C 머지 대기 중 stacked PR — base=feature/issue-7-b2c-academy-connect
- issue #9 생성 + decision 노트 (4 대안 × 2 — 학원 관계 + 코드 정책)

### 2. 코드 (commit 단위)

- C1: AcademyManagementService + AcademyController (create/get/patch) + 3 exception + 3 DTO + AcademyExceptionHandler 매핑 3개 + Academy.rename invariant + findByAcademyIdOrderByCreatedAtDesc repo 메서드
- C2: AcademyCodeService (random alphabet + saveAndFlush + 재시도) + AcademyCodeResponse + AcademyCodeController + OwnerMembershipQueueController (pending 큐)
- C3: AcademyManagementApiTest 10 회귀 + CHANGELOG B3A 섹션

### 3. PR
- PR [#10](https://github.com/seum-running/bkurs-be/pull/10) (feature/issue-9 → feature/issue-7-b2c-academy-connect, stacked) open
- 회귀 **34/34 GREEN**
- CI: B2A → B2B → B2C → 본 PR 순으로 머지되면 자동 실행

# 인수인계 (Handover) — 왜 이렇게 만들었나

| 구현 | 선택 이유 | 대안 / 후속 |
|---|---|---|
| 1인 1학원 정책 | bkurs MVP 의 학원장 1인 가정. UI / 권한 / "내 학원" 선택 흐름 모두 단순화. | B4+ academy_member 테이블 도입 시 분원 / 다중 학원 / 위임 운영자 지원. |
| 다중 active code 허용 | 학원 운영 현실 (분기별/강사용/신학기/보충 등) — 단일 active 제약 시 학원장 UX 깨짐. | 만료 / max_uses / 라벨은 B3 후속 PR 또는 B4 에서 옵션 추가. |
| 32-char alphabet (ambiguous 제외) | 0/O/I/1 혼동 방지 — 학원장이 종이/포스터에 인쇄해 학생에게 전달 시 입력 실수 감소. | URL-safe base32 / base58 검토했으나 학원장이 손으로 적을 때는 영문대문자가 가장 안전. |
| 8자 길이 | 32^8 ≈ 1.1조 조합 → 학원 10만 × 코드 100/학원 = 천만 코드에서도 생일문제 충돌 확률 < 1e-7. 그러나 saveAndFlush + 재시도가 안전망. | 너무 길면 학생 입력 부담. 너무 짧으면 충돌 + 추측. 8자가 sweet spot. |
| saveAndFlush() (NOT save) | save() 는 flush 가 트랜잭션 commit 시점이라 ConstraintViolationException 을 service 안에서 catch 못 함 → 재시도 불가. saveAndFlush 가 즉시 트리거. | 재시도 5회는 안전 마진 — 충돌 확률 무시 가능하므로 실제로 1회 만에 성공. |
| ownership 2계층 (role @PreAuthorize + service academyOwner) | role 만 검증 시 다른 학원장이 우리 코드 deactivate 가능. service 가 code.academyId 와 owner.academy.id 매칭 확인. | 회귀 10번 (`deactivate_otherOwner_403`) 이 명시적 검증. |
| 클래스 레벨 @PreAuthorize | 메서드별 반복 차단 — controller 전체가 OWNER 만 진입. | 일부 endpoint 가 모든 role 허용 필요 시 메서드 레벨로 override. |
| OwnerMembershipQueueController 분리 | AcademyCodeController 와 책임 다름 (code 발급 vs 학원장 운영 화면). 같은 controller 에 묶으면 URL 패턴 혼란 (`/me/codes` vs `/me/memberships`). | B3B 에서 `OwnerDashboardController` 로 통합 검토 가능. |
| Academy.rename(name) invariant | service 가 entity.name = X 직접 mutation 시 blank/null 차단 분산. 도메인 메서드로 강제. | B3B 의 학원 정보 추가 변경 (전화번호 / 주소 / 로고) 도 같은 패턴. |
| 1인 1학원 검증 = findByOwnerAccountId.isEmpty | unique constraint 대신 service 검증 — 향후 1:N 으로 확장 시 schema 변경 없이 정책만 풀 수 있음. | DB 레벨 UNIQUE 도 가능했지만 schema lock-in 회피. |

# 다음 단계 (예정)

- B2A → B2B → B2C → B3A 순으로 사용자 manual merge → CI 자동 실행 → 본 task-log status complete
- B3B 시작 — 강사(TEACHER) 등록 + 학원 소속 + 강사 권한 endpoint
- B3C — 학원 공지 / 알림
- B4 — 플랜 / 결제 / 학원 이관

# 회수 / pitfall (누적)

- `save()` 의 lazy flush 패턴 — DataIntegrityViolationException 을 service 안에서 catch 하려면 saveAndFlush 필수. vault `20-areas/backend/spring/java-spring/pitfalls/save-vs-saveAndFlush-unique-retry.md` 후보.
- 클래스 레벨 @PreAuthorize 가 모든 메서드 강제 — 메서드별 변경 시 override 명시 필요 (의도치 않은 권한 확장 방지).

# 참고

- [[../decisions/decision-academy-management-owner-policy]]
- [[../decisions/decision-academy-connect-membership-flow]]
- [[task-log-b2c-academy-connect]]
