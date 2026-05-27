---
title: "bkurs-fe api-client — stats"
kind: handover
status: stable
created_at: 2026-05-22T01:30:00+09:00
tags: [bkurs-fe, handover, api-client, stats]
related:
  - "[[catalog]]"
  - "[[../../../bkurs-be/handover/domains/stats]]"
home_hub: "[[../../bkurs-fe]]"
---

# stats client

> 학생 7 영역 통계 aggregator. BE [[../../../bkurs-be/handover/domains/stats|stats]] 도메인과 매핑.

## 1. 파일

- `lib/api/stats.ts` — axios 클라이언트
- `lib/hooks/useStats.ts` — react-query wrapper

## 2. 주요 client 메서드

| 메서드 | endpoint |
| --- | --- |
| `statsApi.forStudent()` | `GET /v1/students/me/stats` |
| `statsApi.forTeacher(sid)` | `GET /v1/teachers/me/students/{id}/stats` |
| `statsApi.forOwner(sid)` | `GET /v1/academies/me/students/{id}/stats` |

> 모든 메서드는 `unwrap(res.data)` 로 envelope 풀어 raw data 반환. 에러는 `ApiClientError` throw.

## 3. hook

```
(useStats 후속)
```

query key 컨벤션: `['stats', ...sub]` (예: `['stats', 'mine']`, `['stats', id, 'sub']`). 자세히
[[../decisions/decision-hook-convention]].

## 4. 사용 페이지

- (admin 학생 상세 stats 탭 후속)

## 5. cross-link

- [[catalog]] — 22 client 인덱스
- [[../../../bkurs-be/handover/domains/stats]] — 대응 BE 도메인 (책임 / 엔티티 / endpoint / service 로직)
- [[../modules/lib-layer-guide]] — axios / react-query 컨벤션
