---
title: "bkurs-fe api-client — suhaeng"
kind: handover
status: stable
created_at: 2026-05-22T01:30:00+09:00
tags: [bkurs-fe, handover, api-client, suhaeng]
related:
  - "[[catalog]]"
  - "[[../../../bkurs-be/handover/domains/suhaeng]]"
home_hub: "[[../../bkurs-fe]]"
---

# suhaeng client

> 중등 수행/면담 round (submit / resubmit / feedback / finalize). BE [[../../../bkurs-be/handover/domains/suhaeng|suhaeng]] 도메인과 매핑.

## 1. 파일

- `lib/api/suhaeng.ts` — axios 클라이언트
- `lib/hooks/useSuhaeng.ts` — react-query wrapper

## 2. 주요 client 메서드

| 메서드 | endpoint |
| --- | --- |
| `suhaengApi.listMine()` | `GET /v1/students/me/suhaeng` |
| `suhaengApi.getForStudent(id)` | `GET /v1/students/me/suhaeng/{id}` |
| `suhaengApi.submitFirst(id, answer)` | `POST /v1/students/me/suhaeng/{id}/submit` |
| `suhaengApi.resubmit(id, answer)` | `POST /v1/students/me/suhaeng/{id}/resubmit` |
| `suhaengApi.createByOwner(academyId, req)` | `POST /v1/academies/{id}/suhaeng` |
| `suhaengApi.listByAcademy(academyId)` | `GET /v1/teachers/academies/{id}/suhaeng` |
| `suhaengApi.feedback(roundId, req)` | `POST /v1/teachers/suhaeng-rounds/{id}/feedback` |
| `suhaengApi.finalize(roundId, score)` | `POST /v1/teachers/suhaeng-rounds/{id}/finalize` |

> 모든 메서드는 `unwrap(res.data)` 로 envelope 풀어 raw data 반환. 에러는 `ApiClientError` throw.

## 3. hook

```
useSuhaeng.useMine / useGet / useSubmitFirst / useResubmit / useCreate / useListByAcademy / useFeedback / useFinalize
```

query key 컨벤션: `['suhaeng', ...sub]` (예: `['suhaeng', 'mine']`, `['suhaeng', id, 'sub']`). 자세히
[[../decisions/decision-hook-convention]].

## 4. 사용 페이지

- high-student/suhaeng
- middle-student/suhaeng
- admin/suhaeng (manage)

## 5. cross-link

- [[catalog]] — 22 client 인덱스
- [[../../../bkurs-be/handover/domains/suhaeng]] — 대응 BE 도메인 (책임 / 엔티티 / endpoint / service 로직)
- [[../modules/lib-layer-guide]] — axios / react-query 컨벤션
