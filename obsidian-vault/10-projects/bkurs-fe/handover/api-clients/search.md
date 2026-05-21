---
title: "bkurs-fe api-client — search"
kind: handover
status: stable
created_at: 2026-05-22T01:30:00+09:00
tags: [bkurs-fe, handover, api-client, search]
related:
  - "[[catalog]]"
  - "[[../../../bkurs-be/handover/domains/search]]"
home_hub: "[[../../bkurs-fe]]"
---

# search client

> 5 도메인 통합 LIKE 검색 + role 분기. BE [[../../../bkurs-be/handover/domains/search|search]] 도메인과 매핑.

## 1. 파일

- `lib/api/search.ts` — axios 클라이언트
- `lib/hooks/useSearch.ts` — react-query wrapper

## 2. 주요 client 메서드

| 메서드 | endpoint |
| --- | --- |
| `searchApi.byStudent(q)` | `GET /v1/students/me/search` |
| `searchApi.byTeacherForStudent(sid, q)` | `GET /v1/teachers/me/students/{id}/search` |
| `searchApi.byOwnerForStudent(sid, q)` | `GET /v1/academies/me/students/{id}/search` |

> 모든 메서드는 `unwrap(res.data)` 로 envelope 풀어 raw data 반환. 에러는 `ApiClientError` throw.

## 3. hook

```
(useSearch 후속)
```

query key 컨벤션: `['search', ...sub]` (예: `['search', 'mine']`, `['search', id, 'sub']`). 자세히
[[../decisions/decision-hook-convention]].

## 4. 사용 페이지

- (글로벌 검색 UI 후속)

## 5. cross-link

- [[catalog]] — 22 client 인덱스
- [[../../../bkurs-be/handover/domains/search]] — 대응 BE 도메인 (책임 / 엔티티 / endpoint / service 로직)
- [[../modules/lib-layer-guide]] — axios / react-query 컨벤션
