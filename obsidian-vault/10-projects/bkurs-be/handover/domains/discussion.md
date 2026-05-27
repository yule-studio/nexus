---
title: "도메인 — discussion (토론 + 협업 평가)"
kind: handover
status: stable
created_at: 2026-05-21T23:52:00+09:00
tags: [bkurs-be, handover, domain, discussion]
related:
  - "[[../../bkurs-be]]"
  - "[[record]]"
  - "[[../decisions/decision-discussion-model]]"
home_hub: "[[../../bkurs-be]]"
---

# discussion 도메인

> 강사가 토론 topic 발행 (DEBATE / SEMINAR / QNA), 학생이 입장 (FOR / AGAINST / NEUTRAL) 선택해서
> post (uk upsert), 강사가 0~100 평가. **record bridge → AUTONOMY 카테고리**.

## 1. 책임

- 강사 DiscussionTopic CRUD
- 학생 DiscussionPost upsert (한 topic 1 post / 학생)
- 강사 협업 평가 (0~100) — `evaluate()` 시 record 자동 bridge

## 2. 핵심 엔티티

| 엔티티 | 핵심 |
| --- | --- |
| `DiscussionTopic` | id / courseId / type / title / body |
| `DiscussionPost` | uk(topic_id, account_id) + position / body / evaluation 0~100 (nullable) + evaluatedAt / evaluatedByAccountId |
| `DiscussionType` | DEBATE / SEMINAR / QNA |
| `DiscussionPosition` | FOR / AGAINST / NEUTRAL |

## 3. 패키지 구성

```
discussion/
├── controller/
│   ├── TeacherDiscussionController  # topic CRUD + post 목록 + 평가
│   └── StudentDiscussionController  # topic 목록 + post upsert + 내 post
├── service/                         # DiscussionService — topic + post + evaluate (event publish)
├── domain/
├── dto/{request,response}/
├── enums/                           # DiscussionErrorCode (DIS-)
├── exception/
└── handler/
```

## 4. API endpoint

### TEACHER (6)

| Method | Path | 책임 |
| --- | --- | --- |
| POST | `/api/v1/teachers/me/courses/{courseId}/discussions` | topic 생성 |
| GET | `/api/v1/teachers/me/courses/{courseId}/discussions` | 목록 |
| PATCH/DELETE | `/api/v1/teachers/me/discussions/{topicId}` | |
| GET | `/api/v1/teachers/me/discussions/{topicId}/posts` | 학생 post 목록 |
| POST | `/api/v1/teachers/me/discussion-posts/{postId}/evaluate` | 평가 → event publish |

### STUDENT (2)

| Method | Path | 책임 |
| --- | --- | --- |
| GET | `/api/v1/students/me/discussions` | 내 수업 topic |
| POST | `/api/v1/students/me/discussions/{topicId}/posts` | post upsert |
| GET | `/api/v1/students/me/discussions/{topicId}/my-post` | 내 post |

## 5. Migration

| V | 내용 |
| --- | --- |
| V19 | `discussion_topic` + `discussion_post` (uk + position enum + evaluate audit) |

## 6. 핵심 코드 흐름

### evaluate + record bridge

```
DiscussionService.evaluate(postId, teacherId, score)
  ▼
1. post = postRepo.findById(...)
2. enrollment / teacher 검증
3. post.evaluate(score, teacherId, now)
4. eventPublisher.publishEvent(new DiscussionEvaluatedEvent(
       studentId=post.accountId, postId=post.id, content=post.body))
       ─→ StudentRecordBridgeService.onDiscussion(ev) ─ same TX upsert AUTONOMY 카테고리
```

source-of-truth: `DiscussionPost`. record entry 는 derived. discussion DELETE 시 record 보존 (생기부 영구).

### 협업 (한 topic 안에 다수 학생)

`CourseEnrollmentRepository.findByAccountIdAndStatus(accountId, ACTIVE)` — 본인 enrolled topic 만
볼 수 있음. PR #44 에서 본 메서드 추가.

## 7. 의존

- **의존**: course (FK, enrollment), record (event publish)
- **의존받음**: record bridge listener, stats (참여 카운트), search

## 서비스 로직 (`discussion/service/`)

1 service. 강사 + 학생 + record bridge publish 모두 본 service.

| 클래스 | Tx | 책임 |
| --- | --- | --- |
| `DiscussionService` | `@Transactional` (write) / `readOnly` (조회) | topic CRUD + post upsert + evaluate + AUTONOMY bridge event |

### 핵심 메서드

```java
public DiscussionTopic createByTeacher(Long teacherAccountId, Long courseId, DiscussionType type, String title, String body)
public DiscussionTopic updateByTeacher(Long teacherAccountId, Long topicId, ...)
public void            deleteByTeacher(Long teacherAccountId, Long topicId)
public List<DiscussionTopic> listByTeacherCourse(Long teacherAccountId, Long courseId)
public List<DiscussionTopic> listByStudent(Long studentAccountId)

public Optional<DiscussionPost> findMyPost(Long studentAccountId, Long topicId)
public DiscussionPost submitByStudent(Long studentAccountId, Long topicId, DiscussionPosition position, String body)
                                                          // course enrollment ACTIVE 검증 + uk(topic_id, account_id) upsert
public DiscussionPost evaluateByTeacher(Long teacherAccountId, Long postId, Integer evaluation, String feedback)
                                                          // 강사 권한 + evaluation 0~100 + audit
                                                          // → eventPublisher.publishEvent(new DiscussionEvaluatedEvent(...))
                                                          //   record bridge 가 AUTONOMY 카테고리로 upsert (same TX)
public List<DiscussionPost> listPostsByTeacher(Long teacherAccountId, Long topicId)
```

핵심: 학생 post upsert 검증 — `CourseEnrollmentRepository.findByAccountIdAndStatus(accountId, ACTIVE)` 추가 (PR #44).

## cross-link

- [[course]]
- [[record]] — AUTONOMY bridge
- [[../decisions/decision-discussion-model]]
