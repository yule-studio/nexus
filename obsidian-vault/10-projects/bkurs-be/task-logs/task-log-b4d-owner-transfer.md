---
title: "B4D — 학원장 이관 (양측 confirm + audit) 작업 로그"
kind: task-log
project: bkurs-be
session_id: bkurs-be-b4-payment
created_at: 2026-05-20T10:00:00+09:00
agent: engineering-agent/tech-lead
status: in-progress
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-owner-transfer-two-party]]"
  - "[[task-log-b4c-payment-gateway]]"
tags: [task-log, owner-transfer, audit, b4d]
---

# 작업

GitHub issue [#21](https://github.com/seum-running/bkurs-be/issues/21) — 학원장 이관 + 양측 confirm + audit.

# 결정 요약 (인수인계용)

| 결정 | 내용 | 상세 |
|---|---|---|
| 흐름 | 양측 confirm (FROM initiate → TO confirm) | [[../decisions/decision-owner-transfer-two-party]] |
| 데이터 | 별도 academy_owner_transfer 테이블, 모든 row 영구 보존 | audit trail |
| ACTIVE 제한 | 학원당 PENDING 1개 (service 강제) | 경합 차단 |
| TO 검증 | OWNER role + 학원 미보유 (1인 1학원 정책 유지) | confirm 시 재검증 |
| Actor invariant | entity 가 actor 검증 (TO 만 confirm, FROM 만 cancel) | 일방 변경 차단 |

# 진행

## 2026-05-20

### 1. branch / issue
- B4C 머지 대기 중 stacked PR — base=feature/issue-19-b4c-payment-gateway
- issue #21 + decision 노트 (4 대안 × 2 — 흐름 + TO 학원 보유 정책)

### 2. 코드 (commit 단위)
- C1: AcademyOwnerTransfer entity + V10 + OwnerTransferStatus enum + entity invariant (confirm/reject/cancel actor 검증) + Academy.transferTo + Repository
- C2: 6 예외 + OwnerTransferService (initiate TO valid + ACTIVE 1개 + confirm academy ownership 변경 단일 트랜잭션 + reject + cancel)
- C3: 2 DTO + 2 controller (FROM /me/owner-transfers + TO /owner-transfers) + AcademyExceptionHandler 매핑 6
- C4: OwnerTransferApiTest 10 + CHANGELOG B4D

### 3. PR
- PR [#22](https://github.com/seum-running/bkurs-be/pull/22) (feature/issue-21 → feature/issue-19-b4c-payment-gateway, stacked)
- 회귀 **92/92 GREEN** — B4 시리즈 완결

# 인수인계 (Handover) — 왜 이렇게 만들었나

| 구현 | 선택 이유 | 대안 / 후속 |
|---|---|---|
| 별도 transfer 테이블 (audit trail) | SQL 직접 또는 MASTER 강제 변경은 감사 부재 + 양측 동의 없음 → 분쟁 위험. row 영구 보존으로 모든 시도 추적. | 향후 audit 만 별도 logging service 로 분리 가능. |
| 양측 confirm (FROM initiate → TO confirm) | TO 가 모르는 학원 떠안음 차단 + FROM 이 모르는 인계 차단. 동의 명확. | 일방 변경은 보안 사고. MASTER 강제 endpoint 는 비상 (B5+ 별도 PR). |
| entity actor 검증 (confirm/reject 는 TO 만, cancel 은 FROM 만) | service 가 잘못된 actor 호출 시 invariant 위반 — domain 메서드가 강제. | service 만 검증하면 향후 다른 service 가 entity 직접 mutation 시 차단 불가. |
| status 4종 (PENDING/CONFIRMED/REJECTED/CANCELLED) | 모든 결과 row 영구 보존 — REJECTED / CANCELLED 도 회계 / 분쟁 / 운영 추적. | 자동 EXPIRED (TO 응답 없음 7일) 는 B5+ scheduler. |
| 학원당 ACTIVE 1개 (service 강제) | 같은 학원에 PENDING 여러 개 = 경합. confirm 누구 받아야 하나? | DB unique 두면 향후 정책 유연성 부족. service 강제가 정답. |
| TO 학원 미보유 검증 (initiate + confirm 둘 다) | initiate 시점에 미보유였어도 confirm 시점에 보유 가능 (시간 차). 재검증 필수. | 동시 confirm 가능 — race condition 은 unique index 또는 select for update 검토 (B5+). |
| Academy.transferTo(newOwnerId) 도메인 메서드 | service 가 academy.ownerAccountId 직접 mutation 시 invariant 분산. | null 차단 + 향후 transfer 이력 컬럼 추가 시 entity 가 관리. |
| /me/owner-transfers (FROM) vs /owner-transfers (TO) URL prefix | FROM 은 본인 학원 관점, TO 는 본인 받는 관점 — URL 분리로 의미 명확. | 같은 prefix 면 라우팅 모호. /me vs prefix 없음 패턴 일관. |
| Controller 분리 (FromController / ToController) | path prefix 다름 + 책임 다름 (initiate vs confirm). 단일 controller 면 endpoint 7+ 비대화. | 향후 admin/transfer (MASTER 강제) 는 또 다른 controller. |
| confirm 단일 트랜잭션 (transfer + academy 동시 변경) | 부분 실패 시 (transfer CONFIRMED 인데 academy 그대로) = 데이터 불일치. @Transactional 으로 정합성. | atomic 보장. |
| TO 학원 보유 시 confirm 차단 (cancel 후 받기) | 1인 1학원 정책 (B3A) 일관. TO 의 본인 학원 데이터 (학생/강사/구독) 자동 cancel 은 위험. | 명시적 cancel 요구로 사용자 인지 보장. |
| audit row 보존 (academy DELETE 시만 CASCADE) | 정상 운영에서는 academy 삭제 없음 — transfer audit 영구. | 향후 회계 보고 / 감사 응답 / 운영 통계 모두 활용. |

# 다음 단계 (예정)

- B2A → ... → B4D 순으로 사용자 manual merge → CI 자동 실행
- B5 시작 — 학생 프로필 (학교급 분기 / 학년 / 학원 매칭)
- B6 — 수업 / 일정 / 학사 관리

# 회수 / pitfall (누적)

- TO 학원 보유 검증 두 시점 (initiate / confirm) — 시간차 race condition 가능. service select for update 또는 unique index 검토 (B5+).
- transfer audit 보존 정책 — academy DELETE 만 CASCADE, 일반 운영에선 영구.

# 참고

- [[../decisions/decision-owner-transfer-two-party]]
- [[task-log-b4c-payment-gateway]]
