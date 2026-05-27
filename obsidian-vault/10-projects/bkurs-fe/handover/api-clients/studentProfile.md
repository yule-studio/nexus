---
title: "bkurs-fe api-client — studentProfile"
kind: handover
status: stable
created_at: 2026-05-22T01:30:00+09:00
tags: [bkurs-fe, handover, api-client, studentProfile]
related:
  - "[[catalog]]"
  - "[[../../../bkurs-be/handover/domains/student]]"
home_hub: "[[../../bkurs-fe]]"
---

# studentProfile client

> 학생 프로필 upsert / patch / get (본인 + 학원장). BE [[../../../bkurs-be/handover/domains/student|student]] 도메인과 매핑.

## 1. 파일

- `lib/api/studentProfile.ts` — axios 클라이언트
- `lib/hooks/useStudentProfile.ts` — react-query wrapper

## 2. 주요 client 메서드

| 메서드 | endpoint |
| --- | --- |
| `studentProfileApi.getMine()` | `GET /v1/students/me/profile` |
| `studentProfileApi.upsertMine(req)` | `POST /v1/students/me/profile` |
| `studentProfileApi.patchMine(req)` | `PATCH /v1/students/me/profile` |
| `studentProfileApi.getByOwner(studentId)` | `GET /v1/academies/me/students/{id}/profile` |

> 모든 메서드는 `unwrap(res.data)` 로 envelope 풀어 raw data 반환. 에러는 `ApiClientError` throw.

## 3. hook

```
useStudentProfile.useMine / useUpsertMine / usePatchMine / useByOwner
```

query key 컨벤션: `['studentProfile', ...sub]` (예: `['studentProfile', 'mine']`, `['studentProfile', id, 'sub']`). 자세히
[[../decisions/decision-hook-convention]].

## 4. 사용 페이지

- high/middle signup→connect 흐름
- admin/students/detail

## 5. cross-link

- [[catalog]] — 22 client 인덱스
- [[../../../bkurs-be/handover/domains/student]] — 대응 BE 도메인 (책임 / 엔티티 / endpoint / service 로직)
- [[../modules/lib-layer-guide]] — axios / react-query 컨벤션
