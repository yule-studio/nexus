---
title: "bkurs-fe api-client — saenggibu"
kind: handover
status: stable
created_at: 2026-05-22T01:30:00+09:00
tags: [bkurs-fe, handover, api-client, saenggibu]
related:
  - "[[catalog]]"
  - "[[../../../bkurs-be/handover/domains/saenggibu]]"
home_hub: "[[../../bkurs-fe]]"
---

# saenggibu client

> 중등 생기부 + 학기 lock (composite PK). BE [[../../../bkurs-be/handover/domains/saenggibu|saenggibu]] 도메인과 매핑.

## 1. 파일

- `lib/api/saenggibu.ts` — axios 클라이언트
- `lib/hooks/useSaenggibu.ts` — react-query wrapper

## 2. 주요 client 메서드

| 메서드 | endpoint |
| --- | --- |
| `saenggibuApi.upsertMine(academyId, grade, semester, content)` | `POST /v1/students/me/saenggibu` |
| `saenggibuApi.listMine(grade?, semester?)` | `GET /v1/students/me/saenggibu` |
| `saenggibuApi.setLock(academyId, grade, semester)` | `POST /v1/academies/{id}/saenggibu/lock` |
| `saenggibuApi.getLock(academyId, grade, semester)` | `GET /v1/academies/{id}/saenggibu/lock` |
| `saenggibuApi.listByAcademy(academyId, grade?, semester?)` | `GET /v1/teachers/academies/{id}/saenggibu` |

> 모든 메서드는 `unwrap(res.data)` 로 envelope 풀어 raw data 반환. 에러는 `ApiClientError` throw.

## 3. hook

```
useSaenggibu.useUpsertMine / useMyByTerm / useSetLock / useGetLock / useByAcademy
```

query key 컨벤션: `['saenggibu', ...sub]` (예: `['saenggibu', 'mine']`, `['saenggibu', id, 'sub']`). 자세히
[[../decisions/decision-hook-convention]].

## 4. 사용 페이지

- middle-student/record (중등 생기부)

## 5. cross-link

- [[catalog]] — 22 client 인덱스
- [[../../../bkurs-be/handover/domains/saenggibu]] — 대응 BE 도메인 (책임 / 엔티티 / endpoint / service 로직)
- [[../modules/lib-layer-guide]] — axios / react-query 컨벤션
