---
title: "bkurs-fe api-client — academy"
kind: handover
status: stable
created_at: 2026-05-22T01:30:00+09:00
tags: [bkurs-fe, handover, api-client, academy]
related:
  - "[[catalog]]"
  - "[[../../../bkurs-be/handover/domains/academy]]"
home_hub: "[[../../bkurs-fe]]"
---

# academy client

> 학원 + membership + 강사 invite + 이관. BE [[../../../bkurs-be/handover/domains/academy|academy]] 도메인과 매핑.

## 1. 파일

- `lib/api/academy.ts` — axios 클라이언트
- `lib/hooks/useAcademy.ts` — react-query wrapper

## 2. 주요 client 메서드

| 메서드 | endpoint |
| --- | --- |
| `academyApi.getMine()` | `GET /v1/academies/me` |
| `academyApi.patchMine(req)` | `PATCH /v1/academies/me` |
| `academyApi.issueCode()` | `POST /v1/academies/me/codes` |
| `academyApi.connectAsStudent(code)` | `POST /api/v1/auth/connect` |
| `academyApi.connectAsTeacher(token)` | `POST /api/v1/auth/teacher-connect` |
| `academyApi.approveMembership(id)` | `POST /v1/academies/memberships/{id}/approve` |
| `academyApi.listOwnerTransfers()` | `GET /v1/academies/me/owner-transfers` |

> 모든 메서드는 `unwrap(res.data)` 로 envelope 풀어 raw data 반환. 에러는 `ApiClientError` throw.

## 3. hook

```
useAcademy.useMyAcademy / usePatchAcademy / useCodes / useMembership.useQueue/Approve/Reject / useTeacherInvite / useOwnerTransfer
```

query key 컨벤션: `['academy', ...sub]` (예: `['academy', 'mine']`, `['academy', id, 'sub']`). 자세히
[[../decisions/decision-hook-convention]].

## 4. 사용 페이지

- admin/academy
- admin/teachers
- admin/student-approval
- _shared/auth/useStudentConnect

## 5. cross-link

- [[catalog]] — 22 client 인덱스
- [[../../../bkurs-be/handover/domains/academy]] — 대응 BE 도메인 (책임 / 엔티티 / endpoint / service 로직)
- [[../modules/lib-layer-guide]] — axios / react-query 컨벤션
