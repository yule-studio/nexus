---
title: "B4B — 학원 플랜 + 구독 작업 로그"
kind: task-log
project: bkurs-be
session_id: bkurs-be-b4-payment
created_at: 2026-05-20T09:00:00+09:00
agent: engineering-agent/tech-lead
status: in-progress
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-plan-subscription-model]]"
  - "[[task-log-b4a-academy-info]]"
tags: [task-log, plan, subscription, b4b]
---

# 작업

GitHub issue [#17](https://github.com/seum-running/bkurs-be/issues/17) — 학원 플랜 카탈로그 + 학원장 구독.

# 결정 요약 (인수인계용)

| 결정 | 내용 | 상세 |
|---|---|---|
| 데이터 모델 | plan catalog + academy_plan_subscription 분리 | [[../decisions/decision-plan-subscription-model]] |
| 권한 | MASTER catalog CRUD, OWNER subscription | 책임 분리 |
| 결제 연동 | B4C 에서 — 본 PR 은 mock subscribe | |
| ACTIVE 1개 정책 | 학원당 ACTIVE 1개, schema 는 N row 허용 (이력) | service 강제 |
| change | 기존 cancel + 신규 subscribe 단일 트랜잭션 | 정합성 |
| price | INT (KRW 원 단위) | currency 컬럼은 향후 다통화 도입 시 |

# 진행

## 2026-05-20

### 1. branch / issue
- B4A 머지 대기 중 stacked PR — base=feature/issue-15-b4a-academy-info
- issue #17 + decision 노트 (4 대안 × 2)

### 2. 코드 (commit 단위)
- C1: Plan + AcademyPlanSubscription entity + V8 + Status enum + cancel/update invariant
- C2: 4 예외 + PlanService (MASTER CRUD + catalog) + SubscriptionService (subscribe/cancel/change + ACTIVE 1개 강제)
- C3: 5 DTO + 3 controller (Admin MASTER / Catalog 인증사용자 / Subscription OWNER) + PlanExceptionHandler 5 매핑
- C4: PlanSubscriptionApiTest 10 + CHANGELOG B4B

### 3. PR
- PR [#18](https://github.com/seum-running/bkurs-be/pull/18) (feature/issue-17 → feature/issue-15-b4a-academy-info, stacked)
- 회귀 **72/72 GREEN**

# 인수인계 (Handover) — 왜 이렇게 만들었나

| 구현 | 선택 이유 | 대안 / 후속 |
|---|---|---|
| catalog + subscription 분리 | 가격 변경이 기존 구독에 영향 안 미침 (plan_id snapshot). 정산 / 환불 / 운영 이력 추적. | academy 컬럼 직접 (current_plan_id) 은 이력 없음 → 정산 불가. |
| `subscription.status` enum (ACTIVE/CANCELLED/EXPIRED) | 운영 상태 명확. 같은 학원에 ACTIVE 1 + CANCELLED N (이력) + EXPIRED N (만료). | 자동 EXPIRED 전이는 scheduler (B4C+). 본 PR 은 expires_at 기록만. |
| ACTIVE 1개 정책 service 검증 (schema 는 N row 허용) | 같은 학원에 ACTIVE 2개는 비즈니스 의미 충돌, schema unique 두면 향후 정책 변경 (단계적 마이그레이션) 불가. service 강제로 유연성. | DB partial unique index (PostgreSQL) 가능하지만 MySQL 8 는 미지원 — service 강제가 정답. |
| change = cancel + subscribe 단일 트랜잭션 | 부분 실패 시 (cancel 됐는데 신규 subscribe 실패) 학원이 plan 없는 상태 — 위험. @Transactional 으로 정합성. | atomic API 한 호출 — 두 endpoint 호출보다 안전. |
| price INT (KRW 원) | 결제 정밀도 충분 (소수점 없음). currency 컬럼 없으면 KRW 고정. | 다통화 (USD 등) 도입 시 currency + amount BIGINT (cents) 컬럼 검토. |
| billing_cycle_months INT (1 또는 12) | 월간 / 연간 두 옵션만 — 운영 단순. | 다른 cycle (반년 등) 도입 시 enum 또는 별도 컬럼. |
| expires_at 자동 계산 (started_at + cycle * 30일) | 30일 정확하지 않지만 결제 cycle 추정으로 충분. 실제 만료는 B4C 의 결제 갱신이 결정. | 실제 cycle 은 결제 시스템 (포트원 등) 의 정확한 다음 결제일 사용. |
| `PlanService.listActive()` 인증 사용자 누구나 | 학원장이 가입 직후에도 카탈로그 보고 구독 결정 가능. STUDENT 도 어떤 학원 플랜인지 확인 가능 (투명성). | active only — 운영자가 비활성화한 legacy plan 은 안 보임. |
| `@PreAuthorize MASTER` 클래스 레벨 | admin endpoint 모두 동일 권한 → 메서드별 반복 회피. | 메서드별 다른 role 필요 시 클래스 레벨 제거. |
| /admin/plans (REST prefix) | 일반 endpoint 와 prefix 분리 — 향후 admin 화면 (vue/react) 통합 시 base path 분리. | /api/v1/master/* 같은 명명도 가능. /admin 이 더 일반적. |
| PlanExceptionHandler 도메인 한정 (com.bkurs.plan) | 다른 도메인 (academy/announcement) 의 동일 status code 매핑과 충돌 없음. envelope 일관성. | global handler 통합은 B4 끝나고 별도 cleanup PR 검토. |
| billingCycleMonths 와 plan 의 monthly_price 분리 | 가격이 monthly 기준 — cycle 에 따라 곱셈 (B4C 결제 시점에 계산). plan 에 cycle 별 가격 N 컬럼 두면 비대화. | 할인 (연간 결제 10% off) 같은 정책은 별도 promotion 테이블 (B4C+). |

# 다음 단계 (예정)

- B2A → ... → B4B 순으로 사용자 manual merge → CI 자동 실행
- B4C 시작 — 결제 게이트웨이 (포트원 / 토스 / 네이버 추상화 + mock) 도입 → subscribe 가 실제 결제 호출
- B4D — 학원장 이관 (owner_account_id 변경)

# 회수 / pitfall (누적)

- ACTIVE row 의 unique 제약 (DB 레벨) — MySQL 8 partial unique 미지원 → service 강제. vault 함정 노트 후보 (DB-specific 제약 부재 시 application 강제 패턴).

# 참고

- [[../decisions/decision-plan-subscription-model]]
- [[task-log-b4a-academy-info]]
