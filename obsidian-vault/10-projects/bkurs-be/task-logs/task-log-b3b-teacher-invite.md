---
title: "B3B — 강사 invite + 학원 assignment 작업 로그"
kind: task-log
project: bkurs-be
session_id: bkurs-be-b3-academy
created_at: 2026-05-20T07:30:00+09:00
agent: engineering-agent/tech-lead
status: in-progress
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-teacher-invite-assignment]]"
  - "[[../decisions/decision-academy-management-owner-policy]]"
  - "[[task-log-b3a-academy-management]]"
tags: [task-log, teacher, invite, assignment, b3b]
---

# 작업

GitHub issue [#11](https://github.com/seum-running/bkurs-be/issues/11) — 강사 invite + 학원 assignment + 강사 endpoint.

# 결정 요약 (인수인계용)

| 결정 | 내용 | 상세 |
|---|---|---|
| 가입 흐름 | invite token 발급 → 강사 일반 signup → token 으로 학원 연결 (2 단계) | [[../decisions/decision-teacher-invite-assignment]] |
| 데이터 모델 | 별도 `teacher_academy_assignment` 테이블 (학생 membership 과 분리) | 학생 PENDING_APPROVE flow 없음, 즉시 ACTIVE |
| invite | token 32-byte base64url, 7일 만료, 1회용 | 보안 + 운영 단순 |
| 권한 | role(@PreAuthorize) + ownership(service) 2계층 — B3A 패턴 재사용 | |
| 만료/used 응답 통합 | TeacherInviteUnusableException 단일 409 — 정보 노출 최소화 | "유효한 token 인데 만료" vs "유효하지 않은 token" 구분 못 함 |

# 진행

## 2026-05-20

### 1. branch / issue
- B3A 머지 대기 중 stacked PR — base=feature/issue-9-b3a-academy-management
- issue #11 생성 + decision 노트 (4 대안 × 2 — 가입 흐름 + 데이터 모델)

### 2. 코드 (commit 단위)
- C1: TeacherInvite + TeacherAcademyAssignment entity + V5 + 2 Repository + Status enum + consume/remove invariant 도메인 메서드
- C2: 5 예외 + TeacherInviteService (issue/list/connect) + TeacherAssignmentService (getMyActive/listOwnedActive/remove) — 32-byte SecureRandom base64url token + 7일 TTL
- C3: 3 DTO + 3 controller (TeacherInviteController OWNER / TeacherConnectController role-by-service / TeacherAssignmentController 메서드별 @PreAuthorize) + AcademyExceptionHandler 매핑 5
- C4: TeacherInviteApiTest 10 회귀 + 4 기존 테스트 cleanup 보강 (teacher_*Repository 추가) + CHANGELOG B3B 섹션

### 3. PR
- PR [#12](https://github.com/seum-running/bkurs-be/pull/12) (feature/issue-11 → feature/issue-9-b3a-academy-management, stacked)
- 회귀 **44/44 GREEN**

# 인수인계 (Handover) — 왜 이렇게 만들었나

| 구현 | 선택 이유 | 대안 / 후속 |
|---|---|---|
| invite + signup 2 단계 (NOT signup-with-invite) | 강사 본인이 비번 설정, invite token 은 학원-연결 권한만, 자격 분리. UX 약간 복잡하지만 보안 우선. | 1 단계 통합은 운영 피드백 후 별도 PR. signup endpoint 분기는 DTO/validation/응답 모두 분기로 복잡도 증가. |
| 별도 `teacher_academy_assignment` 테이블 | 학생 membership 의 PENDING_APPROVE flow 와 의미 충돌 방지. 학원장 큐 query 가 role 필터 없이 단순. | B4 학원 이관 시 `academy_member` 통합 검토. |
| 32-byte base64url token (NOT JWT) | DB lookup 으로 검증 — self-contained 클레임 불필요. JWT 면 토큰 자체에 invite 정보 노출. | refresh token (B2B) 과 같은 패턴 — opaque random + DB hash 없음 (refresh 는 SHA-256 hash, invite 는 평문 OK 짧은 TTL + 1회용). |
| invite TTL 7일 | 학원장이 강사에게 전달 → 강사가 가입 → connect 까지의 운영 시간 마진. | 짧으면 운영 불편, 길면 보안 위험. 7일이 운영 균형. |
| 1회용 (used_at + used_by_account_id) | 같은 token 으로 여러 강사 연결 차단 — invite 의 의도 (특정 강사 1명) 와 일치. | 다중 사용 (max_uses) 은 별도 정책 결정 후 컬럼 추가. |
| 만료 / used 응답 통합 (409 teacher_invite_unusable) | 탈취자에게 "유효한 token 인데 만료" vs "유효하지 않은 token" 구분 못 함 → enumeration 공격 방지. reason 필드는 운영 로그용. | refresh reuse detection 과 같은 보안 원칙. |
| TeacherInvite.consumeBy(accountId, now) invariant | service 가 invite.usedAt = now 직접 mutation 시 invariant (이미 used / expired) 검증 분산. 도메인 메서드로 강제. | B3 의 다른 도메인 (TeacherAssignment.removeBy) 도 같은 패턴. |
| TeacherAcademyAssignment 같은 (account, academy) 여러 row 허용 | REMOVED 후 재초대 / 학원 이동 / 재가입 시나리오 — 이력 보존. | UNIQUE 안 둠 — schema 유연성. |
| 강사 1명 = ACTIVE 학원 1개 (service 검증 X, schema 는 N:M) | B3B 정책 — 강사가 동시 다중 학원 운영은 운영 정책 미정. 향후 풀 가능. | service `findFirstByAccountIdAndStatus` 가 ACTIVE 1개만 가정 — 정책 변경 시 서비스만 수정. |
| TeacherAssignmentController 메서드별 @PreAuthorize | 같은 controller 에 강사 / 학원장 endpoint 혼재 (path prefix 다름) → 클래스 레벨 불가, 메서드별 명시. | 분리 controller 도 가능했으나 도메인 응집 우선. |
| AuthApi/Refresh/Connect/AcademyMgmt 4 테스트 cleanup 보강 | V5 FK 추가로 기존 cleanup 깨짐. 5+ FK 누적 패턴 → BaseAcademyTest 추출 후보. | 함정 노트 후보지만 같은 회수 (`save-vs-saveAndFlush` 와 동일 패턴) 라 단독 노트 보류. |

# 다음 단계 (예정)

- B2A → B2B → B2C → B3A → B3B 순으로 사용자 manual merge → CI 자동 실행
- B3C 시작 — 학원 공지 (announcement entity + 학원장이 작성 / 학생·강사 조회 + read 추적)
- B4 — 플랜 + 결제 + 학원 이관 + 학원 정보 (전화/주소/로고)

# 회수 / pitfall (누적)

- V5 FK 누적으로 cleanup 의존 grow → 5+ FK 누적 시 BaseAcademyTest 헬퍼 추출 후보. B3C 추가 시 결정.
- @PreAuthorize 메서드 레벨 / 클래스 레벨 혼용 — 같은 controller 안에서 role 다른 endpoint 가 있으면 메서드 레벨, 통일이면 클래스 레벨.

# 참고

- [[../decisions/decision-teacher-invite-assignment]]
- [[../decisions/decision-academy-management-owner-policy]]
- [[task-log-b3a-academy-management]]
