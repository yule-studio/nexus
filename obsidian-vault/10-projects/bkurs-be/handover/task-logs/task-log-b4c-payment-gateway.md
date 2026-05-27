---
title: "B4C — 결제 게이트웨이 + Payment 이력 + 환불 작업 로그"
kind: task-log
project: bkurs-be
session_id: bkurs-be-b4-payment
created_at: 2026-05-20T09:30:00+09:00
agent: engineering-agent/tech-lead
status: in-progress
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-payment-gateway-abstraction]]"
  - "[[../decisions/decision-plan-subscription-model]]"
  - "[[task-log-b4b-plan-subscription]]"
tags: [task-log, payment, gateway, refund, b4c]
---

# 작업

GitHub issue [#19](https://github.com/seum-running/bkurs-be/issues/19) — 결제 게이트웨이 추상화 + Payment 이력 + 환불.

# 결정 요약 (인수인계용)

| 결정 | 내용 | 상세 |
|---|---|---|
| 추상화 | `PaymentGateway` interface (Strategy) | [[../decisions/decision-payment-gateway-abstraction]] |
| 기본 구현 | `MockPaymentGateway` (amount > 0 success / < 0 fail 시뮬) | 실제 게이트웨이 B5+ |
| Payment ↔ Subscription | 1:N (월정 갱신 / 부분 환불) | FK CASCADE |
| status | PENDING → SUCCESS/FAILED → REFUNDED | 도메인 메서드 invariant |
| 결제 실패 시 | SubscriptionFailedException(402) + @Transactional rollback | subscription row 도 함께 정리 |
| free plan | gateway skip + amount=0 SUCCESS row | UX 단순 |

# 진행

## 2026-05-20

### 1. branch / issue
- B4B 머지 대기 중 stacked PR — base=feature/issue-17-b4b-plan-subscription
- issue #19 + decision 노트 (4 대안 × 2 — 추상화 + Payment 관계)

### 2. 코드 (commit 단위)
- C1: Payment entity + PaymentStatus enum + V9 + 도메인 메서드 (markSuccess/markFailed/markRefunded) + Repository
- C2: PaymentGateway interface (ChargeResult/RefundResult record + 정적 헬퍼 ok/fail) + MockPaymentGateway + SubscriptionFailedException + PaymentService
- C3: SubscriptionService 확장 (chargeOrFail / cancelAndRefund / listMyPayments) + SubscriptionController.POST refund + PaymentController (OWNER /me/payments + MASTER /admin/payments) + PaymentResponse DTO + PlanExceptionHandler 402 매핑
- C4: PaymentApiTest 10 + PlanSubscription cleanup 보강 + CHANGELOG B4C

### 3. PR
- PR [#20](https://github.com/seum-running/bkurs-be/pull/20) (feature/issue-19 → feature/issue-17-b4b-plan-subscription, stacked)
- 회귀 **82/82 GREEN**

# 인수인계 (Handover) — 왜 이렇게 만들었나

| 구현 | 선택 이유 | 대안 / 후속 |
|---|---|---|
| `PaymentGateway` interface (Strategy) | 게이트웨이 교체 (포트원 → 토스) = 구현체 swap. SDK 변경이 비즈니스 로직과 격리. | 추상 클래스 상속은 향후 multi-gateway / fallback 어려움. interface 가 정답. |
| `MockPaymentGateway` 기본 구현 | 로컬 / 테스트 / 초기 운영 가능. 실제 게이트웨이 추가 시 @ConditionalOnProperty / @Profile swap. | 다중 gateway 동시 운영 (fallback) 시 `@Primary` + `List<PaymentGateway>` 주입. |
| `Payment` row 의 PENDING → SUCCESS/FAILED → REFUNDED 도메인 메서드 | service 가 entity.status 직접 mutation 시 invariant 분산. 도메인 메서드로 강제 (markSuccess 가 PENDING 아닐 때 throw). | requireStatus invariant 가 모든 도메인 (membership / subscription / payment) 의 공통 패턴. |
| ChargeResult/RefundResult 정적 헬퍼 ok/fail | record 자동 accessor `success()` 와 충돌 회피 — 정적 메서드 이름 `success()` 못 씀. | success / failed 가 자연스럽지만 record 제약. ok / fail 이 호출 측 readability 도 OK. |
| 결제 실패 시 SubscriptionFailedException → @Transactional rollback | subscription row + payment row 둘 다 rollback — 학원 입장에서 "구독 안 됨" 정확. noRollbackFor 안 두는 게 정답. | refresh token (B2B) 의 reuse 는 noRollbackFor 가 정답이었지만, 결제 실패는 반대 — rollback 이 정확. |
| 402 Payment Required | 결제 실패의 표준 HTTP status. 401/403/409 와 구분. | 일반 4xx 보다 명확. 클라가 "결제 화면 재진입" 분기 명확. |
| free plan (price=0) gateway skip | 무료 plan 도 gateway 호출하면 0원 결제 → UX 어색 + gateway 일부는 0원 거부. service 가 amount=0 분기 + zero-amount SUCCESS row. | 별도 free plan flag 도 가능했지만 price=0 으로 판단이 단순. |
| Payment.gateway 컬럼 (gateway 이름) | 같은 학원이 시점 별로 다른 gateway 사용 가능 (포트원 → 토스 이전). 감사 / 정산 시 필수. | NULL 두면 마이그레이션 시 추적 불가. NOT NULL 강제. |
| gateway_tx_id 별도 컬럼 | 외부 시스템 ID — 환불 / 분쟁 조회 시 필수. nullable (FAILED 는 ID 없을 수 있음). | 외부 시스템과 우리 row id 매핑 필수. |
| SubscriptionService.cancelAndRefund 단일 트랜잭션 | cancel 만 하고 refund 별도 endpoint 면 부분 실패 시 학원이 "취소됐는데 환불 안 됨" 상태. atomic. | refund 가 gateway 실패해도 payment 는 SUCCESS 유지 — alert 후 수동 환불 운영. |
| /admin/payments MASTER 전체 (감사) | 운영자가 회계 / 정산 / 분쟁 조회 필요. role 분리. | 향후 admin 화면 (vue/react) 에서 필터 / 검색 / export 확장. |
| /me/payments OWNER 본인 학원만 across 모든 구독 | 학원장이 본인 학원 결제 이력 (구독 N 개 누적) 한 곳에서 확인. | 구독 단위 (`/subscription/{id}/payments`) 도 가능하지만 학원장 화면은 통합 뷰가 자연스러움. |

# 다음 단계 (예정)

- B2A → ... → B4C 순으로 사용자 manual merge → CI 자동 실행
- B4D 시작 — 학원장 이관 (owner_account_id 변경 + 감사 로그 + 양측 confirm)
- B5+ — 실제 게이트웨이 (포트원/토스) 구현 + webhook + 영수증 + 부분 환불

# 회수 / pitfall (누적)

- **record 정적 헬퍼 vs accessor 이름 충돌** — record 의 필드 `success` 가 자동으로 `success()` accessor 생성 → 정적 메서드 `success()` 불가. ok/fail 패턴이 안전. vault 함정 노트 후보 (record 제약 일반).

# 참고

- [[../decisions/decision-payment-gateway-abstraction]]
- [[../decisions/decision-plan-subscription-model]]
- [[task-log-b4b-plan-subscription]]
