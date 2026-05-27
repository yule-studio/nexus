---
title: "도메인 — plan (플랜 / 구독 / 결제 / 환불)"
kind: handover
status: stable
created_at: 2026-05-21T23:45:00+09:00
tags: [bkurs-be, handover, domain, plan, payment]
related:
  - "[[../../bkurs-be]]"
  - "[[../external-gateways]]"
  - "[[../decisions/decision-plan-subscription-model]]"
  - "[[../decisions/decision-payment-gateway-abstraction]]"
home_hub: "[[../../bkurs-be]]"
---

# plan 도메인

> 학원 단위 SaaS 구독. **MASTER 가 catalog 관리**, OWNER 가 구독 / 결제 / 환불. PaymentGateway 가
> 추상화되어 있어 Mock ↔ PortOne 실 PG swap 가능 (사이클 2 PR #101).

## 1. 책임

- MASTER: Plan catalog CRUD
- OWNER: `AcademyPlanSubscription` (1 학원 1 ACTIVE 구독) — subscribe / cancel / change
- OWNER: `Payment` row (구독 결제 / 환불) — PaymentGateway 가 외부 PG 호출
- 미납 알림 batch ([[batch]]) 가 본 도메인 조회

## 2. 핵심 엔티티

| 엔티티 | 핵심 |
| --- | --- |
| `Plan` | id / name / monthlyPriceCents / billingCycle / features json |
| `AcademyPlanSubscription` | academyId / planId / status (ACTIVE/CANCELLED/EXPIRED) + startedAt / endsAt — 학원당 1 ACTIVE invariant |
| `Payment` | id / subscriptionId / amountCents / status (PENDING/SUCCESS/FAILED/REFUNDED) / pgTxnId / paidAt — invariant: 환불은 SUCCESS 만 가능 |
| `SubscriptionStatus` | enum |

## 3. 패키지 구성

```
plan/
├── controller/
│   ├── AdminPlanController          # /api/v1/admin/plans — MASTER CRUD
│   ├── PlanCatalogController        # /api/v1/plans — (auth required) 카탈로그 조회
│   ├── SubscriptionController       # /api/v1/academies/me/subscription — OWNER subscribe/cancel/change/refund
│   └── PaymentController            # /api/v1/academies/me/payments + /api/v1/admin/payments
├── service/                         # PlanService / SubscriptionService / PaymentService
├── domain/                          # Plan / Subscription / SubscriptionStatus + Repository
├── payment/                         # Payment + PaymentRepository (sub-package — entity 분리)
├── gateway/
│   ├── PaymentGateway               # interface
│   ├── MockPaymentGateway           # @Profile("!prod")
│   ├── PortOnePaymentGateway        # @Profile("prod") — PR #101
│   └── PortOneProperties            # @ConfigurationProperties("portone.*")
├── dto/{request,response}/
├── enums/                           # PlanErrorCode (PLA-), PaymentErrorCode (PAY-)
├── exception/
└── handler/
```

## 4. API endpoint

### Plan catalog (MASTER)

| Method | Path | 책임 |
| --- | --- | --- |
| POST | `/api/v1/admin/plans` | 생성 |
| PATCH | `/api/v1/admin/plans/{id}` | 수정 |
| GET | `/api/v1/admin/plans` | MASTER 전체 목록 |
| GET | `/api/v1/plans` | (인증된 모두) 카탈로그 |

### Subscription (OWNER)

| Method | Path | 책임 |
| --- | --- | --- |
| POST | `/api/v1/academies/me/subscription` | 구독 + 결제 호출 |
| GET | `/api/v1/academies/me/subscription` | 본인 구독 |
| POST | `/api/v1/academies/me/subscription/cancel` | 취소 (다음 주기 EXPIRED) |
| POST | `/api/v1/academies/me/subscription/change` | 플랜 변경 |
| POST | `/api/v1/academies/me/subscription/refund` | 환불 — `SubscriptionService.cancelAndRefund` |

### Payment 조회

| Method | Path | role | 책임 |
| --- | --- | --- | --- |
| GET | `/api/v1/academies/me/payments` | OWNER | 본인 학원 결제 내역 |
| GET | `/api/v1/admin/payments` | MASTER | 전체 결제 |

## 5. Migration

| V | 내용 |
| --- | --- |
| V8 | `plan` + `academy_plan_subscription` (status enum + uk academy_id + endsAt) |
| V9 | `payment` (status enum + pg_txn_id + invariant 환불 = SUCCESS 만) |

## 6. 핵심 코드 흐름

### subscribe (PaymentGateway 호출)

```
SubscriptionService.subscribe(academyId, planId)
  ▼
1. existsActiveByAcademyId(academyId) ─ 1 ACTIVE 검증
2. Plan plan = planRepo.findById(planId) ─ 없으면 NotFound
3. Payment payment = Payment.builder().status(PENDING).build() ─ 사전 row
4. paymentRepo.save(payment)
5. PaymentGateway.charge(ChargeRequest(plan.priceCents, academyId, payment.id))
     ├── SUCCESS → payment.markPaid(pgTxnId, paidAt)
     └── FAILED  → payment.markFailed(reason) + throw PaymentFailedException
6. AcademyPlanSubscription sub = new ACTIVE subscription
7. messagingPublisher.publish(SUBSCRIPTION_STARTED, ...) ─ async
```

### refund

```
SubscriptionService.cancelAndRefund(subId)
  ▼
1. sub.status = ACTIVE 검증
2. lastPayment.status = SUCCESS 검증 ─ 아니면 402 PaymentNotRefundable
3. PaymentGateway.refund(pgTxnId, amount)
4. payment.markRefunded(refundedAt) + sub.cancel()
```

402 매핑: PaymentErrorCode 의 `PAY-4020 PAYMENT_REQUIRED`.

## 7. PaymentGateway 추상화

interface:

```java
public interface PaymentGateway {
    PaymentResult charge(ChargeRequest req);    // PG 호출 + idempotent key
    PaymentResult refund(String pgTxnId, long amountCents);
}
```

| 환경 | 구현 |
| --- | --- |
| local / test | `MockPaymentGateway` — 95% SUCCESS / 5% FAILED 시뮬 |
| prod (사이클 2) | `PortOnePaymentGateway` — REST adapter + webhook |

`PortOneProperties` (`@ConfigurationProperties(prefix = "portone")`): `imp-key` / `imp-secret` /
`merchant-id`.

## 8. 의존

- **의존**: auth, academy
- **의존받음**: batch.OverduePaymentAlertJobConfig (미납 학원 조회), dashboard (학원장 결제 지표)

## 9. 회수

| 회수 | 해결 |
| --- | --- |
| Refund 가 SUCCESS 아닌 payment 에 호출됨 | `cancelAndRefund` 사전 status 검증 + 402 |
| PG 응답 webhook 처리 후속 | 별도 endpoint `/webhook/portone` (signature 검증) + payment row 동기화 — 사이클 후속 |

## 서비스 로직 (`plan/service/`)

2 service. `SubscriptionService` 가 `PaymentGateway` 호출의 핵심 진입점.

| 클래스 | Tx | 책임 |
| --- | --- | --- |
| `PlanService` | `@Transactional` | MASTER catalog CRUD |
| `SubscriptionService` | `@Transactional` | 학원 1 ACTIVE invariant + PaymentGateway 호출 + 환불 |

### 핵심 메서드

```java
// PlanService
public Plan create(String name, Integer monthlyPrice, Integer maxStudents, Integer maxTeachers)
public Plan update(Long planId, String name, Integer monthlyPrice, ...)
public List<Plan> listAll()
public List<Plan> listActive()
public Plan getById(Long planId)

// SubscriptionService
public AcademyPlanSubscription subscribe(Long ownerAccountId, Long planId, Integer billingCycleMonths)
   // 1. existsActiveByAcademyId → 충돌 시 AlreadyActiveSubscriptionException
   // 2. Payment PENDING row 저장
   // 3. paymentGateway.charge(req) → SUCCESS/FAILED
   //    - SUCCESS → payment.markPaid, ACTIVE subscription INSERT
   //    - FAILED  → payment.markFailed + throw PaymentFailedException
   // 4. messagingPublisher.publish(SUBSCRIPTION_STARTED, ...) async
public AcademyPlanSubscription getMyActive(Long ownerAccountId)
public AcademyPlanSubscription cancelMine(Long ownerAccountId)            // status CANCELLED (다음 cycle EXPIRED)
public AcademyPlanSubscription change(Long ownerAccountId, Long newPlanId, Integer billingCycleMonths)
public AcademyPlanSubscription cancelAndRefund(Long ownerAccountId, String reason)
   // lastPayment.status=SUCCESS 검증 → paymentGateway.refund(pgTxnId, amount) → markRefunded + sub.cancel
```

PaymentGateway 추상화 / 실 구현 swap 은 [[../modules/external-gateways]] §7.

## cross-link

- [[academy]] — academy_plan_subscription FK
- [[batch]] — 미납 알림 job
- [[../external-gateways#payment]] — PortOne / Mock 상세
- [[../decisions/decision-plan-subscription-model]]
- [[../decisions/decision-payment-gateway-abstraction]]
