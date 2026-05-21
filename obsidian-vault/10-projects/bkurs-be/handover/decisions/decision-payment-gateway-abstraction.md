---
title: "B4C 결정 — 결제 게이트웨이 추상화 + Payment 이력 + 환불"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T09:30:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, payment, gateway, refund, b4c]
related:
  - "[[../bkurs-be]]"
  - "[[decision-plan-subscription-model]]"
home_hub: bkurs-be
---

# B4C 결정 — 결제 게이트웨이 추상화 + Payment 이력 + 환불

## 결론

1. **`PaymentGateway` interface** — `charge(request) → result`, `refund(request) → result`. 구현체는 Mock / 포트원 / 토스 등 교체 가능.
2. **`MockPaymentGateway` 구현** — B4C 기본. amount > 0 이면 항상 SUCCESS, amount < 0 이면 FAILED (테스트 시뮬레이션). 실제 게이트웨이는 B5+.
3. **`Payment` entity 별도 테이블** — subscription_id FK + status (PENDING/SUCCESS/FAILED/REFUNDED) + amount + gateway / gateway_tx_id (감사 trail).
4. **subscribe flow 결제 통합** — `charge` 호출 → SUCCESS 시 subscription ACTIVE 부여, FAILED 시 subscription 생성 안 함.
5. **환불 flow** — `POST /me/subscription/refund` → 결제 환불 + subscription CANCELLED 단일 트랜잭션.

## 왜 필요한가

- B4B subscribe 가 mock — 실제 운영에서는 결제 결과 받아야 ACTIVE 부여.
- 결제 게이트웨이 (한국: 포트원/토스/네이버페이) 가 다양 — 단일 interface 로 교체 가능해야 운영 유연성.
- 환불 추적 (영수증 / 회계 / 분쟁) 위해 Payment row 보존 필수.

## 안 하면 뭐가 문제

- 결제 호출 안 하면 무료 구독 — 비즈니스 모델 깨짐.
- gateway 직접 호출하면 다른 gateway 도입 시 코드 전체 변경.
- 결제 이력 없으면 영수증 / 환불 / 정산 모두 불가능.

## 4 대안 비교 — 추상화 방식

### A. Service 가 게이트웨이 SDK 직접 호출 — 기각

- 단점: SDK 변경 = 비즈니스 로직 변경. 테스트 시 mock 어려움.
- **기각**.

### B. PaymentGateway interface + 구현체 분리 (Strategy) — **채택**

- 장점: 게이트웨이 교체 = 구현체 swap. 테스트는 Mock 구현 사용.
- 단점: 새 게이트웨이 추가 시 구현체 추가 (정상 비용).
- **채택 사유**: Spring `@Profile` / `@ConditionalOnProperty` 로 환경별 swap 가능.

### C. PaymentGateway 추상 클래스 + 구현 상속

- 단점: 다중 상속 불가 → 향후 게이트웨이 조합 (multi-gateway / fallback) 어려움.
- **기각**.

### D. 외부 결제 microservice 분리

- 단점: B4 범위 초과 — service mesh / API 호출 / 인증 등 인프라 부담.
- **기각**.

## 4 대안 비교 — Payment ↔ Subscription 관계

### a. Payment 가 subscription_id FK 1:N — **채택**

- 장점: 1 구독에 N 결제 (월정 갱신 / 부분 환불). 정상 관계.
- 단점: 구독 cancel 시 payment 도 정리 필요 (CASCADE 또는 manual).

### b. Subscription 에 last_payment_id 컬럼

- 단점: 이력 못 봄. 회계 query 불가.
- **기각**.

### c. Payment 만 저장 + subscription 도출 (event sourcing)

- 단점: B4B 의 subscription 모델 폐기. 큰 변화.
- **기각**.

### d. 외부 결제 시스템에 모두 위임 (우리 DB 미저장)

- **기각**: B4C 결정 a 의 정반대.

## 테이블 설계

### `payment`

```sql
CREATE TABLE payment (
  id              BIGINT AUTO_INCREMENT PRIMARY KEY,
  subscription_id BIGINT NOT NULL,
  gateway         VARCHAR(32) NOT NULL,           -- "mock" / "portone" / "toss"
  gateway_tx_id   VARCHAR(128) NULL,              -- gateway 가 발행한 transaction ID
  amount          INT NOT NULL,                   -- KRW 원 단위, 음수 안 됨 (CHECK)
  status          VARCHAR(32) NOT NULL,           -- PENDING / SUCCESS / FAILED / REFUNDED
  failure_reason  VARCHAR(500) NULL,
  created_at      DATETIME(6) NOT NULL,
  completed_at    DATETIME(6) NULL,               -- SUCCESS / FAILED 시점
  refunded_at     DATETIME(6) NULL,
  CONSTRAINT fk_payment_subscription
    FOREIGN KEY (subscription_id) REFERENCES academy_plan_subscription(id) ON DELETE CASCADE,
  KEY idx_payment_subscription (subscription_id),
  KEY idx_payment_status (status)
);
```

## PaymentGateway interface

```java
public interface PaymentGateway {
    String name();   // "mock", "portone", "toss"

    ChargeResult charge(ChargeRequest req);
    RefundResult refund(RefundRequest req);

    record ChargeRequest(Long subscriptionId, int amount, String description) {}
    record ChargeResult(boolean success, String gatewayTxId, String failureReason) {}

    record RefundRequest(String gatewayTxId, int amount, String reason) {}
    record RefundResult(boolean success, String failureReason) {}
}
```

## endpoint 설계

| method | path | role | 검증 |
|---|---|---|---|
| POST | /api/v1/academies/me/subscription | OWNER | 기존 + charge 호출 |
| POST | /api/v1/academies/me/subscription/refund | OWNER | active sub + 환불 |
| GET | /api/v1/academies/me/payments | OWNER | 본인 결제 이력 |
| GET | /api/v1/admin/payments | MASTER | 전체 결제 |

## 회귀 라인

`PaymentApiTest`:
1. OWNER subscribe → 결제 SUCCESS + subscription ACTIVE + payment row
2. 무료 plan (price=0) subscribe → 결제 SKIP + ACTIVE (정책)
3. 결제 실패 시뮬레이션 → 결제 FAILED + subscription 생성 안 됨
4. OWNER GET /me/payments → 본인 이력 1 row
5. OWNER refund → subscription CANCELLED + payment REFUNDED
6. STUDENT GET /me/payments → 403
7. 다른 학원장 GET → 본인 학원만 (격리)
8. MASTER GET /admin/payments → 전체
9. 환불 후 재구독 가능 → 신규 subscription + 신규 payment
10. payment.amount = plan.monthlyPrice * billingCycleMonths 검증

## 관련

- [[decision-plan-subscription-model]] — 본 결정의 상위 (subscribe / cancel flow 확장)
- 후속: B4D (학원장 이관), B5+ (실제 게이트웨이 구현 추가)
