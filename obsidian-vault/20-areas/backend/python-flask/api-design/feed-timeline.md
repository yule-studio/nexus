---
title: "피드 / 타임라인 — Flask"
kind: knowledge
status: current
created_at: 2026-05-26T17:45:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-flask, api-design, recipe, feed, timeline]
---

# 피드 / 타임라인 — Flask

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/feed-timeline|↗ Java 원본]]**

> Fan-out on Read 기본 + Celery 기반 Hybrid Push.

---

## 1-3. 무엇을 / 전략 / DB 스키마

(Java 원본 [[../../spring/java-spring/api-design/feed-timeline#1|§1, §2, §3]] 와 동일)

---

## 4. Feed 조회 (Fan-out on Read)

```python
# app/repositories/feed_repo.py
from sqlalchemy import text


def get_feed(user_id: str, cursor_created: datetime | None,
             cursor_id: str | None, limit: int) -> list[dict]:
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
    rows = db.session.execute(sql, {
        "user_id": user_id,
        "cursor_created": cursor_created,
        "cursor_id": cursor_id,
        "limit": limit,
    }).mappings().all()
    return [dict(r) for r in rows]
```

---

## 5. 좋아요 토글 — atomic UPDATE

```python
from sqlalchemy import text
from sqlalchemy.exc import IntegrityError


def toggle_like(post_id: str, user_id: str) -> bool:
    """returns: True if liked, False if unliked."""
    existed = db.session.execute(
        text("SELECT 1 FROM post_likes WHERE post_id = :pid AND user_id = :uid"),
        {"pid": post_id, "uid": user_id},
    ).first() is not None

    if existed:
        db.session.execute(
            text("DELETE FROM post_likes WHERE post_id = :pid AND user_id = :uid"),
            {"pid": post_id, "uid": user_id},
        )
        db.session.execute(
            text("UPDATE posts SET like_count = like_count - 1 WHERE id = :pid"),
            {"pid": post_id},
        )
    else:
        try:
            db.session.execute(
                text("INSERT INTO post_likes (post_id, user_id, created_at) "
                     "VALUES (:pid, :uid, now())"),
                {"pid": post_id, "uid": user_id},
            )
            db.session.execute(
                text("UPDATE posts SET like_count = like_count + 1 WHERE id = :pid"),
                {"pid": post_id},
            )
        except IntegrityError:
            # 동시 클릭 — 이미 존재
            pass
    db.session.commit()
    return not existed
```

---

## 6. Hybrid Fan-out (VIP) — Celery

```python
@shared_task
def fan_out_to_followers(post_id: str, author_id: str) -> None:
    followers = follow_repo.followers_of(author_id)
    feed_cache_repo.batch_insert(followers, post_id, datetime.now(timezone.utc))


# SQLAlchemy event — post 작성 후
@event.listens_for(db.session, "after_commit")
def _trigger_fanout(session):
    for obj in session.new:
        if isinstance(obj, Post):
            author = user_repo.find_by_id(obj.author_id)
            if author and author.follower_count > 10_000:
                fan_out_to_followers.delay(obj.id, author.id)
```

> **Python 한정**: SQLAlchemy `after_commit` 가 `session.new` 비어있을 수 있음 — pending objects 가 commit 후 expunge 됨. **별도 list buffer** 또는 `before_commit` 에서 캡처.

---

## 7. 트렌딩

```sql
-- Java 원본 §7 와 동일
SELECT id, title,
       (like_count + comment_count * 2)
       * exp(-extract(epoch from (now() - created_at)) / 86400.0) AS score
FROM posts
WHERE status = 'PUBLISHED'
  AND created_at > now() - INTERVAL '7 days'
ORDER BY score DESC LIMIT 50;
```

---

## 8. Controller

```python
@feed_bp.get("/feed")
@jwt_required()
def my_feed():
    cursor = request.args.get("cursor")
    limit = min(int(request.args.get("limit", 20)), 50)
    user_id = get_jwt_identity()

    items = feed_service.get_feed(user_id, cursor, limit)
    return ok("조회", {"items": items})


@feed_bp.post("/posts/<post_id>/like")
@jwt_required()
def like(post_id: str):
    active = toggle_like(post_id, get_jwt_identity())
    return ok("좋아요" if active else "취소", {"active": active})
```

---

## 9. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/feed-timeline#9 함정 모음|§9]] 의 10 함정 그대로)

### Python/Flask 한정

### 함정 11 — `db.session.execute(text(...))` raw SQL injection
parameter binding (`{":pid": ...}`) 항상 사용. f-string 으로 SQL 만들지 말 것.

### 함정 12 — Celery `.delay()` 가 task arg type 직렬화 못 함
datetime / Decimal / 도메인 객체 그대로 못 보냄. **str / int / dict 만**.

---

## 10. 운영 체크리스트

(Java 원본 §10 그대로)

---

## 11. 관련

- [[../../spring/java-spring/api-design/feed-timeline|↗ Java 원본]]
- [[recommendation]]
- [[api-design|↑ hub]]
