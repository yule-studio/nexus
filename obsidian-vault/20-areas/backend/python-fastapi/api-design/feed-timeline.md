---
title: "피드 / 타임라인 — FastAPI"
kind: knowledge
status: current
created_at: 2026-05-27T00:15:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, api-design, recipe, feed]
---

# 피드 / 타임라인 — FastAPI

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/feed-timeline|↗ Java 원본]]** · **[[../../python-flask/api-design/feed-timeline|↗ Flask 변형]]**

---

## 1-3-7. 무엇을 / 전략 / DB / 트렌딩 SQL

(Java 원본 [[../../spring/java-spring/api-design/feed-timeline#1|§1, §2, §3, §7]] 와 동일)

---

## 4. Feed 조회 (async)

```python
# app/repositories/feed_repo.py
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession


async def get_feed(
    db: AsyncSession, user_id: int,
    cursor_created: datetime | None, cursor_id: str | None, limit: int,
) -> list[dict]:
    sql = text("""
        SELECT p.id, p.author_id AS authorId, p.title, p.content,
               p.like_count AS likeCount, p.comment_count AS commentCount,
               p.created_at AS createdAt
        FROM posts p
        WHERE p.status = 'PUBLISHED'
          AND p.author_id IN (
              SELECT followee_id FROM follows WHERE follower_id = :user_id
              UNION SELECT CAST(:user_id AS CHAR(26))
          )
          AND (:cursor_created IS NULL
               OR (p.created_at, p.id) < (:cursor_created, :cursor_id))
        ORDER BY p.created_at DESC, p.id DESC
        LIMIT :limit
    """)
    result = await db.execute(sql, {
        "user_id": user_id,
        "cursor_created": cursor_created,
        "cursor_id": cursor_id,
        "limit": limit,
    })
    return [dict(r) for r in result.mappings().all()]
```

---

## 5. 좋아요 토글 — atomic UPDATE

```python
from sqlalchemy.exc import IntegrityError


async def toggle_like(db: AsyncSession, post_id: int, user_id: int) -> bool:
    existed = (await db.execute(
        text("SELECT 1 FROM post_likes WHERE post_id = :pid AND user_id = :uid"),
        {"pid": post_id, "uid": user_id},
    )).first() is not None

    if existed:
        await db.execute(
            text("DELETE FROM post_likes WHERE post_id = :pid AND user_id = :uid"),
            {"pid": post_id, "uid": user_id},
        )
        await db.execute(
            text("UPDATE posts SET like_count = like_count - 1 WHERE id = :pid"),
            {"pid": post_id},
        )
    else:
        try:
            await db.execute(
                text("INSERT INTO post_likes (post_id, user_id, created_at) "
                     "VALUES (:pid, :uid, now())"),
                {"pid": post_id, "uid": user_id},
            )
            await db.execute(
                text("UPDATE posts SET like_count = like_count + 1 WHERE id = :pid"),
                {"pid": post_id},
            )
        except IntegrityError:
            await db.rollback()
            # 동시 클릭

    await db.commit()
    return not existed
```

---

## 6. Hybrid Fan-out (VIP) — Celery (sync) 또는 BackgroundTasks (간단)

```python
from fastapi import BackgroundTasks


@router.post("/posts")
async def create_post(
    payload: CreatePostRequest,
    user: CurrentUserDep, db: SessionDep,
    bg: BackgroundTasks,
):
    post = Post(owner_id=user.id, ...)
    db.add(post)
    await db.commit()

    if user.follower_count > 10_000:
        # 간단 — BackgroundTasks (process 안)
        bg.add_task(_fan_out_to_followers, post.id, user.id)
        # 또는 — Celery (다른 process)
        # fan_out_to_followers_task.delay(post.id, user.id)

    return CommonResponse.success(ResponseCode.OK, "작성 완료", post)


async def _fan_out_to_followers(post_id: int, author_id: int) -> None:
    # 별도 db session 필요
    async with AsyncSessionLocal() as db:
        followers = await follow_repo.followers_of(db, author_id)
        await feed_cache_repo.batch_insert(db, followers, post_id)
```

> **FastAPI BackgroundTasks vs Celery**:
> - **BackgroundTasks**: same process, async, 가벼움, fire-and-forget. 작은 작업.
> - **Celery**: 별도 process, retry / monitoring / 분산. 크고 중요한 작업.

---

## 8. Controller

```python
@router.get("/feed", response_model=CommonResponse[FeedResponse])
async def my_feed(
    user: CurrentUserDep, db: SessionDep,
    cursor: str | None = None,
    limit: int = Query(20, ge=1, le=50),
):
    items = await feed_service.get_feed(db, user.id, cursor, limit)
    return CommonResponse.success(ResponseCode.OK, "조회", FeedResponse(items=items))


@router.post("/posts/{post_id}/like")
async def like(post_id: int, user: CurrentUserDep, db: SessionDep):
    active = await toggle_like(db, post_id, user.id)
    return CommonResponse.success(
        ResponseCode.OK,
        "좋아요" if active else "취소",
        {"active": active},
    )
```

---

## 9. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/feed-timeline#9 함정 모음|§9]] + Flask 변형 함정 그대로)

### FastAPI 한정

### 함정 11 — BackgroundTasks 가 process restart 시 사라짐
docker restart / deploy 시 in-flight task 손실. **중요한 작업은 Celery**.

### 함정 12 — `bg.add_task(async_fn, ...)` 안에서 `db.session`
새 session 필요 — `async with AsyncSessionLocal() as db:`. main request session 공유 X.

---

## 10. 운영 체크리스트

(Java 원본 §10 + Python)

---

## 11. 관련

- [[../../spring/java-spring/api-design/feed-timeline|↗ Java 원본]]
- [[../../python-flask/api-design/feed-timeline|↗ Flask 변형]]
- [[recommendation]]
- [[api-design|↑ hub]]
