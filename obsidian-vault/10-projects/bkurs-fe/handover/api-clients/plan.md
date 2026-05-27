---
title: "bkurs-fe api-client — plan"
kind: handover
status: stable
created_at: 2026-05-22T01:30:00+09:00
tags: [bkurs-fe, handover, api-client, plan]
related:
  - "[[catalog]]"
  - "[[../../../bkurs-be/handover/domains/plan]]"
home_hub: "[[../../bkurs-fe]]"
---

# plan client

> 플랜 catalog + 구독 + 결제 + 환불. BE [[../../../bkurs-be/handover/domains/plan|plan]] 도메인과 매핑.

## 1. 파일

- `lib/api/plan.ts` — axios 클라이언트
- `lib/hooks/usePlan.ts` — react-query wrapper

## 2. 주요 client 메서드

| 메서드 | endpoint |
| --- | --- |
| `planApi.listMaster()` | `GET /v1/admin/plans` |
| `planApi.listCatalog()` | `GET /v1/plans` |
| `planApi.subscribe(planId)` | `POST /v1/academies/me/subscription` |
| `planApi.cancelMine()` | `POST /v1/academies/me/subscription/cancel` |
| `planApi.changeMine(planId)` | `POST /v1/academies/me/subscription/change` |
| `planApi.refundMine()` | `POST /v1/academies/me/subscription/refund` |
| `planApi.myPayments()` | `GET /v1/academies/me/payments` |
| `planApi.adminPayments()` | `GET /v1/admin/payments` |

> 모든 메서드는 `unwrap(res.data)` 로 envelope 풀어 raw data 반환. 에러는 `ApiClientError` throw.

## 3. hook

```
usePlan.useMaster / useCatalog / useSubscribe / useCancel / useChange / useRefund / useMyPayments / useAdminPayments
```

query key 컨벤션: `['plan', ...sub]` (예: `['plan', 'mine']`, `['plan', id, 'sub']`). 자세히
[[../decisions/decision-hook-convention]].

## 4. 사용 페이지

- master/billing
- admin/billing

## 5. cross-link

- [[catalog]] — 22 client 인덱스
- [[../../../bkurs-be/handover/domains/plan]] — 대응 BE 도메인 (책임 / 엔티티 / endpoint / service 로직)
- [[../modules/lib-layer-guide]] — axios / react-query 컨벤션
