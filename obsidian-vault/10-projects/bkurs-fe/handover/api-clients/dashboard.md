---
title: "bkurs-fe api-client — dashboard"
kind: handover
status: stable
created_at: 2026-05-22T01:30:00+09:00
tags: [bkurs-fe, handover, api-client, dashboard]
related:
  - "[[catalog]]"
  - "[[../../../bkurs-be/handover/domains/dashboard]]"
home_hub: "[[../../bkurs-fe]]"
---

# dashboard client

> 강사 (수업별 5 지표) / 학원장 (학원 9 지표 rollup). BE [[../../../bkurs-be/handover/domains/dashboard|dashboard]] 도메인과 매핑.

## 1. 파일

- `lib/api/dashboard.ts` — axios 클라이언트
- `lib/hooks/useDashboard.ts` — react-query wrapper

## 2. 주요 client 메서드

| 메서드 | endpoint |
| --- | --- |
| `dashboardApi.teacher()` | `GET /v1/teachers/me/dashboard` |
| `dashboardApi.owner()` | `GET /v1/academies/me/dashboard` |

> 모든 메서드는 `unwrap(res.data)` 로 envelope 풀어 raw data 반환. 에러는 `ApiClientError` throw.

## 3. hook

```
useDashboard.useTeacher / useOwner
```

query key 컨벤션: `['dashboard', ...sub]` (예: `['dashboard', 'mine']`, `['dashboard', id, 'sub']`). 자세히
[[../decisions/decision-hook-convention]].

## 4. 사용 페이지

- admin/dashboard
- (teacher 대시보드 후속)

## 5. cross-link

- [[catalog]] — 22 client 인덱스
- [[../../../bkurs-be/handover/domains/dashboard]] — 대응 BE 도메인 (책임 / 엔티티 / endpoint / service 로직)
- [[../modules/lib-layer-guide]] — axios / react-query 컨벤션
