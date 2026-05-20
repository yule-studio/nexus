---
title: "B4B 결정 — 학원 플랜 + 구독 (catalog + subscription 분리, 결제는 B4C)"
kind: decision
project: bkurs-be
status: decided
created_at: 2026-05-20T09:00:00+09:00
agent: engineering-agent/tech-lead
tags: [decision, plan, subscription, b4b]
related:
  - "[[../bkurs-be]]"
  - "[[decision-academy-management-owner-policy]]"
  - "[[decision-academy-info-fields]]"
home_hub: bkurs-be
---

# B4B 결정 — 플랜 + 구독

## 결론

1. **`plan` 카탈로그 테이블** + **`academy_plan_subscription` 구독 row** 분리. MASTER 가 plan CRUD, OWNER 가 subscription.
2. **subscription status**: ACTIVE / CANCELLED / EXPIRED. cancel 은 즉시 종료 (refund 정책은 B4C+).
3. **결제 연동 X (B4C)** — 본 PR 은 구독 상태 추적 + expires_at 기록만. mock subscribe.
4. **현재 ACTIVE subscription 1개** 정책 — 같은 학원에 2개 active 금지. change 는 기존 cancel + 신규 active 1 트랜잭션.

## 왜 필요한가

- 학원이 요금제 가입해야 수업 / 학생 정원 / 강사 정원 제한 적용 가능.
- 운영자 (MASTER) 가 플랜 카탈로그 관리 — 가격 변경 / 신규 플랜 출시.
- 학원장이 본인 학원의 구독 상태 / 변경 가능.
- 결제 연동 (B4C) 가 큰 작업이라 분리 — B4B 는 구독 모델만 먼저.

## 안 하면 뭐가 문제

- 플랜 없으면 정원 / 기능 제한 못 함 → 무제한 가입 + 운영 비용 폭증.
- 구독 endpoint 없으면 학원장이 운영자에게 요청 → 운영 부담 + 자동화 불가.
- catalog + subscription 분리 안 하면 가격 변경이 기존 구독에 영향 → 운영 사고.

## 4 대안 비교 — 데이터 모델

### A. plan 컬럼을 academy 에 직접 (current_plan_id) — 기각

- 단점: 구독 이력 없음. 가격 변경 / 정산 / 환불 추적 불가.
- **기각**.

### B. academy_plan_subscription 별도 테이블 — **채택**

- 장점: row 단위 이력 — 구독 시점 / 만료 / 변경 모두 추적. 정산 / 환불 query 가능.
- 단점: "현재 ACTIVE 구독" query 시 status 필터 + 정렬 — 인덱스 `(academy_id, status)`.
- **채택 사유**: 정규화 + 운영 추적 + 환불.

### C. event sourcing (subscription_event 만 저장)

- 단점: 모든 query 가 event replay → 성능. snapshot 도입 추가 복잡도.
- **기각**: B4B 범위 초과.

### D. 외부 결제 시스템에만 의존 (DB 미저장)

- 단점: 결제 시스템 의존성 → 우리 DB 와 동기화 어려움 + 인증 / 정산 query 불가능.
- **기각**.

## 4 대안 비교 — 권한 모델

### a. MASTER 만 plan CRUD, OWNER 만 subscription — **채택**

- 장점: 책임 분리. 운영자가 카탈로그 관리, 학원장이 구독 결정.
- 단점: MASTER 계정 운영 도구 필요 — 현재 admin 화면 없음 (API 만 노출).
- **채택**: MVP 정책.

### b. OWNER 도 custom plan 가능

- 단점: 가격 협상 / 운영 부담 폭증. MVP 단계 부적합.
- **기각**.

### c. MASTER 가 subscription 도 강제 변경 가능

- 단점: 학원장 동의 없이 변경 = 운영 분쟁.
- **결정**: B4D 학원장 이관 같은 admin 권한 별도 PR.

## 테이블 설계

### `plan`

```sql
CREATE TABLE plan (
  id              BIGINT AUTO_INCREMENT PRIMARY KEY,
  name            VARCHAR(100) NOT NULL,
  monthly_price   INT NOT NULL,                -- KRW, 원 단위
  max_students    INT NOT NULL,
  max_teachers    INT NOT NULL,
  active          TINYINT(1) NOT NULL DEFAULT 1,
  created_at      DATETIME(6) NOT NULL,
  updated_at      DATETIME(6) NOT NULL,
  CONSTRAINT uk_plan_name UNIQUE (name)        -- 같은 이름 중복 방지
);
```

### `academy_plan_subscription`

```sql
CREATE TABLE academy_plan_subscription (
  id                  BIGINT AUTO_INCREMENT PRIMARY KEY,
  academy_id          BIGINT NOT NULL,
  plan_id             BIGINT NOT NULL,
  status              VARCHAR(32) NOT NULL,   -- ACTIVE / CANCELLED / EXPIRED
  billing_cycle_months INT NOT NULL,           -- 1 or 12
  started_at          DATETIME(6) NOT NULL,
  expires_at          DATETIME(6) NOT NULL,
  cancelled_at        DATETIME(6) NULL,
  CONSTRAINT fk_sub_academy FOREIGN KEY (academy_id) REFERENCES academy(id) ON DELETE CASCADE,
  CONSTRAINT fk_sub_plan FOREIGN KEY (plan_id) REFERENCES plan(id),
  KEY idx_sub_academy_status (academy_id, status)
);
```

## endpoint 설계

| method | path | role | 검증 |
|---|---|---|---|
| POST | /api/v1/admin/plans | MASTER | 권한만 |
| PATCH | /api/v1/admin/plans/{id} | MASTER | 권한만 |
| GET | /api/v1/admin/plans | MASTER | 권한만 |
| GET | /api/v1/plans | 인증된 모든 사용자 | active 플랜만 |
| POST | /api/v1/academies/me/subscription | OWNER | 본인 학원 + active subscription 없음 |
| GET | /api/v1/academies/me/subscription | OWNER | 본인 학원 |
| POST | /api/v1/academies/me/subscription/cancel | OWNER | 본인 학원 active sub |
| POST | /api/v1/academies/me/subscription/change | OWNER | 본인 학원 active sub → cancel + 신규 |

## 회귀 라인

`PlanSubscriptionApiTest`:
1. MASTER 플랜 생성 → 201
2. STUDENT 플랜 생성 → 403
3. GET /plans → active 카탈로그 (모든 인증 사용자)
4. OWNER subscribe → 201 + ACTIVE + expires_at = started_at + 1 month
5. OWNER 중복 subscribe → 409
6. OWNER GET subscription → ACTIVE row
7. OWNER cancel → status CANCELLED + cancelled_at
8. OWNER change (다른 plan) → 기존 row CANCELLED + 신규 ACTIVE
9. 미존재 plan → 404
10. inactive plan subscribe → 409 plan_inactive

## 관련

- [[decision-academy-management-owner-policy]] — OWNER 권한 매트릭스
- [[decision-academy-info-fields]] — 동일 PATCH /me 패턴 재사용
- 후속: B4C (결제 게이트웨이) — 본 결정의 subscribe 가 실제 결제 호출하도록 확장
