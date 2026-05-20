---
title: "B2C — 학원 코드 connect + 학원장 승인/거부 + 상태 전이 작업 로그"
kind: task-log
project: bkurs-be
session_id: bkurs-be-b2-auth
created_at: 2026-05-20T05:30:00+09:00
agent: engineering-agent/tech-lead
status: in-progress
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-academy-connect-membership-flow]]"
  - "[[../decisions/decision-account-single-table-role-status]]"
  - "[[task-log-b2b-refresh-cors]]"
tags: [task-log, academy, membership, status-transition, b2c]
---

# 작업

GitHub issue [#7](https://github.com/seum-running/bkurs-be/issues/7) — 학생 학원 코드 입력 + 학원장 승인/거부 + 상태 전이.

# 결정 요약 (인수인계용)

| 결정 | 내용 | 상세 |
|---|---|---|
| membership 모델 | account 컬럼 X, `student_academy_membership` 별도 테이블 | [[../decisions/decision-academy-connect-membership-flow]] |
| 상태 트랙 | account.status 와 membership.status 이중 — 사용자 관점 vs 학원 관점 | account.status 는 JWT 클레임 / 화면 분기 / membership.status 는 학원장 큐 |
| 권한 검증 | role(@PreAuthorize) + ownership(service layer) 2계층 | role 만 검증 시 다른 학원장이 우리 학원 학생 승인 가능 |
| 학원 모델 | 학원장 1명 = 학원 1개 (B2C), 다중 운영자 / 강사 등록 B3+ | minimal 시작, B3 에서 multi-academy / multi-owner 확장 |
| code 정책 | 학원당 단일 active code, 만료 / max_uses X (B3+) | minimal 시작 |

# 진행

## 2026-05-20

### 1. branch / issue

- B2B (#6) merge 대기 중 stacked PR — branch base=feature/issue-5-b2b-refresh-cors
- issue #7 생성 (assignee codwithyc, label enhancement, audit 섹션에 결정 노트 링크)
- decision 노트 작성 (academy-connect-membership-flow, 4 대안 비교 + 테이블 설계 + 권한 정책 + 회귀 라인)

### 2. 코드 (commit 단위)

- C1: Academy + AcademyCode + StudentAcademyMembership entity + V4 마이그레이션 + MembershipStatus enum + 3 Repository. transitionTo / approveBy / rejectBy invariant 메서드.
- C2: 6 exception (AcademyCodeNotFound/CodeInactive/MembershipAlreadyExists/MembershipNotOwned/MembershipNotFound/NotStudent) + AcademyConnectService.connect + MembershipService.approve/reject. @Transactional + ownership 검증.
- C3: 3 DTO (ConnectRequest/RejectRequest/MembershipResponse) + ConnectController(/auth/connect) + MembershipController(/academies/memberships/{id}/approve|reject) + AcademyExceptionHandler 7 매핑 + SecurityConfig @EnableMethodSecurity. @PreAuthorize("hasRole('OWNER')").
- C4: ConnectApprovalApiTest 8 회귀 + AuthApiTest/RefreshTokenApiTest cleanup 보강 (FK 의존 순서) + CHANGELOG B2C 섹션.

### 3. PR

- PR [#8](https://github.com/seum-running/bkurs-be/pull/8) (feature/issue-7 → feature/issue-5-b2b-refresh-cors, stacked) open
- 회귀 **24/24 GREEN** (smoke 4 + Auth 5 + Refresh 6 + CORS 1 + ConnectApproval 8)
- CI: B2A → B2B → 본 PR 순으로 머지되면 자동 실행

# 인수인계 (Handover) — 왜 이렇게 만들었나

| 구현 | 선택 이유 | 대안 / 후속 |
|---|---|---|
| `Academy.owner_account_id` 단일 컬럼 | B2C 는 학원장 1:1, 다중 운영자 (강사 운영 권한 / 학원장 부재 시 위임) 는 운영 정책 명확 후 B3+. 지금 over-design 하면 데이터 모델 변경 비용. | B3 에서 `academy_member` 테이블 (account + academy + role) 도입 가능. |
| `academy_code.active` boolean (만료 / max_uses 없음) | B2C 의 핵심은 connect flow — code 정책은 학원장 화면 (B3+) 에서 완성. 만료 / 사용 횟수 / 1회용 code 는 별도 결정. | `expires_at` / `max_uses` / `used_count` 컬럼은 V5 (B3) 에서. |
| `student_academy_membership` 의 status enum 4종 | PENDING_APPROVE / ACTIVE / REJECTED 는 immediate flow. INACTIVE 는 B3+ 학원 이동 / 종료 시 — schema 안정성을 위해 미리 enum 정의. | 같은 (account, academy) UNIQUE 안 둠 — REJECTED 후 재신청 가능. 운영 정책에 따라 cooldown 결정. |
| `approveBy(ownerId)` / `rejectBy(ownerId, reason)` 도메인 메서드 | service 가 entity.status = ACTIVE 직접 mutation 하면 invariant (PENDING_APPROVE 만 처리 가능) 검증 분산. 도메인 메서드로 강제. | 같은 패턴이 B3 의 학원 정보 변경 / 강사 등록 endpoint 에서도 재사용. |
| account.status 와 membership.status 이중 트랙 + @Transactional | account.status 가 JWT 클레임 → 클라가 화면 분기 (Connect / Pending / Active) 즉시 가능. membership.status 가 학원장 큐 인덱스. | 둘 중 하나만 있으면 학생 관점 또는 학원장 관점 한쪽이 N+1 query 필요. |
| `@PreAuthorize("hasRole('OWNER')")` + service ownership 검증 (2계층) | role 만 검증 시 다른 학원장이 우리 학원 학생 승인 가능 = 보안 사고. role 1차 + ownership 2차 로 같은 role 안에서도 차단. | 회귀 테스트 7번 (`approve_byOtherOwner_403`) 이 명시적 검증. |
| `AcademyExceptionHandler` 의 `AccessDeniedException` 처리 | @PreAuthorize 의 default 403 응답 body 가 비어있어 클라가 reason 알 수 없음. 명시적 매핑으로 일관된 envelope. | global handler 통합은 B2 끝나고 별도 cleanup PR 검토. |
| `IllegalStateException` → 409 | entity invariant (이미 ACTIVE 인 row 재처리) 위반은 비즈니스 의미 충돌 → 409 가 정확. 500 으로 두면 운영 알람 false positive. | 다른 entity 의 invariant 위반도 같은 매핑 → 글로벌 handler 후보. |
| Test cleanup 순서 명시 (membership → code → academy → refresh → account) | V4 의 FK 가 누적될수록 cleanup 순서 의존 grow → 명시적 helper 또는 `@DirtiesContext` 검토 필요. | 회귀 5+ 누적 시 `TestDataCleaner` 컴포넌트 추출 검토. |
| `TransactionTemplate` 으로 setup 데이터 작성 | @SpringBootTest 의 controller 호출 (다른 트랜잭션) 과 entity @PrePersist 가 충돌 없이 동작하려면 명시적 트랜잭션 필요. | `@Sql` annotation 대안 — 그러나 builder 패턴 활용 못 함. |

# 다음 단계 (예정)

- B2A (#4) → B2B (#6) → B2C (#8) 순으로 사용자 manual merge → CI 자동 실행 → 본 task-log status complete
- B3 시작 — Academy 도메인 (학원 생성 / 강사 등록 / 학원 코드 발급·재발급·만료 / 학원 정보 변경 endpoint)
- vault 본 task-log status: in-progress → complete (사용자 merge 후)

# 회수 / pitfall (누적)

- V4 의 FK (academy.owner_account_id → account, membership.account_id → account) 가 기존 테스트의 `accountRepository.deleteAll()` 를 깨뜨림 → cleanup 순서 (FK 의존 역순) 의식적 관리 필요. 회귀 누적 시 `TestDataCleaner` 헬퍼 추출 후보.
- @PreAuthorize 의 default 403 body 가 비어있음 (Spring Security 의 `AccessDeniedHandler` default) → controller 별 ExceptionHandler 에서 명시적 매핑.

# 참고

- [[../decisions/decision-academy-connect-membership-flow]]
- [[../decisions/decision-account-single-table-role-status]]
- [[task-log-b2b-refresh-cors]]
