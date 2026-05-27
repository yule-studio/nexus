---
title: "리뷰 / 평점 — FastAPI"
kind: knowledge
status: current
created_at: 2026-05-27T00:35:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, api-design, recipe, review]
---

# 리뷰 / 평점 — FastAPI

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/review|↗ Java 원본]]** · **[[../../python-flask/api-design/review|↗ Flask 변형]]**

> async SQLAlchemy + IntegrityError 처리.

---

## 1-2-4-6-7. 무엇을 / DB / 정렬·필터 / moderation / 함정

(Java 원본 [[../../spring/java-spring/api-design/review#1|§1, §2, §4, §6, §7]] 와 동일)

---

## 3. UseCase — async

### 3.1 CreateReviewUseCase

```python
from sqlalchemy import select
from sqlalchemy.exc import IntegrityError
from sqlalchemy.ext.asyncio import AsyncSession

from app.domain.review import Review
from app.domain.order_item import OrderItem
from app.errors import BusinessException, NotFoundException, ForbiddenException, ConflictException


async def create_review(
    db: AsyncSession,
    user_id: int, product_id: str, order_item_id: str,
    rating: int, content: str, image_urls: list[str] | None = None,
) -> Review:
    # 1. 구매 확인
    order_item = await db.get(OrderItem, order_item_id)
    if order_item is None:
        raise NotFoundException("주문 항목")
    if order_item.buyer_id != user_id:
        raise ForbiddenException("본인 주문만 리뷰 가능")
    if order_item.product_id != product_id:
        raise BusinessException(400, "BAD_REQUEST", "주문한 상품이 아닙니다.")
    if not order_item.is_reviewable():
        raise BusinessException(400, "BAD_REQUEST", "리뷰 가능 상태가 아닙니다.")

    # 2. 1인1리뷰
    existing = await db.scalar(
        select(Review).where(Review.user_id == user_id, Review.product_id == product_id)
    )
    if existing:
        raise ConflictException("이미 리뷰를 작성했습니다.")

    # 3. validation
    if not 1 <= rating <= 5:
        raise BusinessException(400, "BAD_REQUEST", "평점은 1~5")
    if not content or len(content) > 2000:
        raise BusinessException(400, "BAD_REQUEST", "리뷰 내용 1~2000자")
    if image_urls and len(image_urls) > 5:
        raise BusinessException(400, "BAD_REQUEST", "이미지 최대 5장")

    review = Review(
        user_id=user_id, product_id=product_id, order_item_id=order_item_id,
        rating=rating, content=content,
    )
    db.add(review)
    try:
        await db.commit()
        await db.refresh(review)
    except IntegrityError:
        await db.rollback()
        raise ConflictException("이미 리뷰를 작성했습니다.")

    # 도메인 이벤트 — BackgroundTasks 또는 Celery
    # (호출자에서 bg.add_task(on_review_created, review.id, product_id, rating))

    return review
```

### 3.2 평점 집계

```python
async def on_review_created(db: AsyncSession, product_id: str) -> None:
    """매 리뷰마다 — 큰 상품은 batch 권장."""
    result = await db.execute(
        text("SELECT COUNT(*) AS count, AVG(rating) AS avg FROM reviews "
             "WHERE product_id = :pid AND status = 'PUBLISHED'"),
        {"pid": product_id},
    )
    row = result.mappings().first()
    await product_stats_repo.upsert(db, product_id, row["count"], row["avg"])
    await db.commit()
```

### 3.3 ToggleHelpfulUseCase

```python
async def toggle_helpful(db: AsyncSession, review_id: int, user_id: int) -> bool:
    review = await db.get(Review, review_id)
    if review is None:
        raise NotFoundException("리뷰")
    if review.user_id == user_id:
        raise BusinessException(400, "BAD_REQUEST", "자기 리뷰에 도움돼요 불가")

    existed = (await db.execute(
        text("SELECT 1 FROM review_helpfuls WHERE review_id = :rid AND user_id = :uid"),
        {"rid": review_id, "uid": user_id},
    )).first() is not None

    if existed:
        await db.execute(
            text("DELETE FROM review_helpfuls WHERE review_id = :rid AND user_id = :uid"),
            {"rid": review_id, "uid": user_id},
        )
        await db.execute(
            text("UPDATE reviews SET helpful_count = helpful_count - 1 WHERE id = :rid"),
            {"rid": review_id},
        )
    else:
        try:
            await db.execute(
                text("INSERT INTO review_helpfuls (review_id, user_id, created_at) "
                     "VALUES (:rid, :uid, now())"),
                {"rid": review_id, "uid": user_id},
            )
            await db.execute(
                text("UPDATE reviews SET helpful_count = helpful_count + 1 WHERE id = :rid"),
                {"rid": review_id},
            )
        except IntegrityError:
            await db.rollback()

    await db.commit()
    return not existed
```

---

## 5. Controller

```python
from pydantic import BaseModel, Field, HttpUrl


class CreateReviewRequest(BaseModel):
    orderItemId: str = Field(min_length=1)
    rating: int = Field(ge=1, le=5)
    content: str = Field(min_length=1, max_length=2000)
    imageUrls: list[HttpUrl] = Field(default_factory=list, max_length=5)


@router.post("/products/{product_id}/reviews", status_code=201)
async def create(
    product_id: str,
    payload: CreateReviewRequest,
    user: CurrentUserDep, db: SessionDep,
    bg: BackgroundTasks,
):
    review = await create_review(
        db, user.id, product_id, payload.orderItemId,
        payload.rating, payload.content,
        [str(u) for u in payload.imageUrls],
    )
    bg.add_task(on_review_created, db, product_id)
    return CommonResponse.success(ResponseCode.OK, "리뷰 작성 완료", {"reviewId": review.id})


@router.post("/reviews/{review_id}/helpful")
async def helpful(review_id: int, user: CurrentUserDep, db: SessionDep):
    active = await toggle_helpful(db, review_id, user.id)
    return CommonResponse.success(
        ResponseCode.OK,
        "도움돼요 표시" if active else "취소",
        {"active": active},
    )
```

> **FastAPI idiom**: `list[HttpUrl]` — pydantic v2 의 URL validator. `max_length=5` 로 list 크기 검증.

---

## 8. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/review#7 함정 모음|§7]] + Flask 변형 함정)

### FastAPI 한정

### 함정 11 — `bg.add_task(async_fn, db, ...)` 의 session 공유
main request 의 `db` session 이 BackgroundTask 안에서 만료. **새 session** (`async with AsyncSessionLocal() as bg_db`) 또는 sync Celery.

### 함정 12 — `HttpUrl` 의 직렬화
pydantic v2 의 `HttpUrl` 은 `Url` 객체. JSON 직렬화 시 자동 str 변환되지만 DB 저장 전 `str(url)` 명시.

---

## 9. 운영 체크리스트

(Java 원본 §8 + Python async)

---

## 10. 관련

- [[../../spring/java-spring/api-design/review|↗ Java 원본]]
- [[../../python-flask/api-design/review|↗ Flask 변형]]
- [[file-upload-s3]]
- [[api-design|↑ hub]]
