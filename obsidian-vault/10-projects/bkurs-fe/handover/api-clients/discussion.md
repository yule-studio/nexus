---
title: "bkurs-fe api-client — discussion"
kind: handover
status: stable
created_at: 2026-05-22T01:30:00+09:00
tags: [bkurs-fe, handover, api-client, discussion]
related:
  - "[[catalog]]"
  - "[[../../../bkurs-be/handover/domains/discussion]]"
home_hub: "[[../../bkurs-fe]]"
---

# discussion client

> 토론 topic + post (uk upsert) + 협업 평가 (BE bridge → record AUTONOMY). BE [[../../../bkurs-be/handover/domains/discussion|discussion]] 도메인과 매핑.

## 1. 파일

- `lib/api/discussion.ts` — axios 클라이언트
- `lib/hooks/useDiscussion.ts` — react-query wrapper

## 2. 주요 client 메서드

| 메서드 | endpoint |
| --- | --- |
| `discussionApi.listMine()` | `GET /v1/students/me/discussions` |
| `discussionApi.myPost(topicId)` | `GET /v1/students/me/discussions/{id}/my-post` |
| `discussionApi.submitPost(topicId, req)` | `POST /v1/students/me/discussions/{id}/posts` |
| `discussionApi.createByCourse(courseId, req)` | `POST /v1/teachers/me/courses/{id}/discussions` |
| `discussionApi.update(id, req)` | `PATCH /v1/teachers/me/discussions/{id}` |
| `discussionApi.listPosts(id)` | `GET /v1/teachers/me/discussions/{id}/posts` |
| `discussionApi.evaluatePost(postId, req)` | `POST /v1/teachers/me/discussion-posts/{id}/evaluate` |

> 모든 메서드는 `unwrap(res.data)` 로 envelope 풀어 raw data 반환. 에러는 `ApiClientError` throw.

## 3. hook

```
useDiscussion.useMine / useMyPost / useSubmitPost / useCreate / useUpdate / useListPosts / useEvaluatePost
```

query key 컨벤션: `['discussion', ...sub]` (예: `['discussion', 'mine']`, `['discussion', id, 'sub']`). 자세히
[[../decisions/decision-hook-convention]].

## 4. 사용 페이지

- high-student/topic
- middle-student/debate

## 5. cross-link

- [[catalog]] — 22 client 인덱스
- [[../../../bkurs-be/handover/domains/discussion]] — 대응 BE 도메인 (책임 / 엔티티 / endpoint / service 로직)
- [[../modules/lib-layer-guide]] — axios / react-query 컨벤션
