---
title: "리뷰 / 평점 — Flask"
kind: knowledge
status: current
created_at: 2026-05-26T22:00:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-flask, api-design, recipe, review, rating]
---

# 리뷰 / 평점 — Flask

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/review|↗ Java 원본]]**

> SQLAlchemy + atomic UPDATE 패턴.

---

## 1. 무엇을 / 2. DB 스키마 / 4. 정렬·필터 / 6. moderation / 7. 함정

(Java 원본 [[../../spring/java-spring/api-design/review#1|§1, §2, §4, §6, §7]] 와 동일)

---

## 3. UseCase

### 3.1 CreateReviewUseCase

```python
# app/services/review_service.py
from sqlalchemy import select
from sqlalchemy.exc import IntegrityError
from sqlalchemy.ext.asyncio import AsyncSession

from app.domain.review import Review
from app.domain.order_item import OrderItem
from app.errors import BusinessException, NotFoundException, ForbiddenException, ConflictException


def create_review(
    db,  # AsyncSession 또는 sync Session
    user_id: int, product_id: str, order_item_id: str,
    rating: int, content: str, image_urls: list[str] | None = None,
) -> Review:
    # 1. 구매 확인
    order_item = db.session.get(OrderItem, order_item_id)
    if order_item is None:
        raise NotFoundException("주문 항목")
    if order_item.buyer_id != user_id:
        raise ForbiddenException("본인 주문만 리뷰 가능")
    if order_item.product_id != product_id:
        raise BusinessException(400, "BAD_REQUEST", "주문한 상품이 아닙니다.")
    if not order_item.is_reviewable():
        raise BusinessException(400, "BAD_REQUEST", "리뷰 가능 상태가 아닙니다 (배송 완료 후).")

    # 2. 1인1리뷰 — DB UNIQUE 가 진실
    if db.session.query(Review).filter_by(user_id=user_id, product_id=product_id).first():
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
    db.session.add(review)
    try:
        db.session.commit()
    except IntegrityError:
        db.session.rollback()
        raise ConflictException("이미 리뷰를 작성했습니다.")

    # 도메인 이벤트 — 별도 background job 또는 Celery
    on_review_created.delay(review.id, product_id, rating)

    return review
```

### 3.2 평점 집계 — Celery task

```python
@shared_task
def on_review_created(review_id: int, product_id: str, rating: int) -> None:
    """매 리뷰마다 전체 재계산은 O(N) — 인기 상품엔 batch 권장."""
    aggregate = db.session.execute(
        text("SELECT COUNT(*) AS count, AVG(rating) AS avg FROM reviews "
             "WHERE product_id = :pid AND status = 'PUBLISHED'"),
        {"pid": product_id},
    ).first()
    product_stats_repo.upsert(product_id, aggregate.count, aggregate.avg)
```

### 3.3 ToggleHelpfulUseCase

```python
from sqlalchemy import text


def toggle_helpful(db, review_id: int, user_id: int) -> bool:
    review = db.session.get(Review, review_id)
    if review is None:
        raise NotFoundException("리뷰")
    if review.user_id == user_id:
        raise BusinessException(400, "BAD_REQUEST", "자기 리뷰에 도움돼요 불가")

    existed = db.session.execute(
        text("SELECT 1 FROM review_helpfuls WHERE review_id = :rid AND user_id = :uid"),
        {"rid": review_id, "uid": user_id},
    ).first() is not None

    if existed:
        db.session.execute(
            text("DELETE FROM review_helpfuls WHERE review_id = :rid AND user_id = :uid"),
            {"rid": review_id, "uid": user_id},
        )
        db.session.execute(
            text("UPDATE reviews SET helpful_count = helpful_count - 1 WHERE id = :rid"),
            {"rid": review_id},
        )
    else:
        try:
            db.session.execute(
                text("INSERT INTO review_helpfuls (review_id, user_id, created_at) "
                     "VALUES (:rid, :uid, now())"),
                {"rid": review_id, "uid": user_id},
            )
            db.session.execute(
                text("UPDATE reviews SET helpful_count = helpful_count + 1 WHERE id = :rid"),
                {"rid": review_id},
            )
        except IntegrityError:
            db.session.rollback()           # 동시 클릭

    db.session.commit()
    return not existed
```

> **함정 (hot key)**: 베스트 리뷰는 도움돼요가 분 단위 폭주. UPDATE 직렬화 = lock 대기. **Redis INCR 로 카운터 + 1분마다 DB sync** 옵션 ([[cache-redis]] 참고).

### 3.4 ReportReviewUseCase

```python
AUTO_HIDE_THRESHOLD = 5


def report_review(db, review_id: int, reporter_id: int, reason: str, detail: str | None) -> None:
    review = db.session.get(Review, review_id)
    if review is None:
        raise NotFoundException("리뷰")

    try:
        db.session.add(ReviewReport(
            review_id=review_id, reporter_id=reporter_id,
            reason=reason, detail=detail, status="PENDING",
        ))
        db.session.commit()
    except IntegrityError:
        db.session.rollback()
        raise ConflictException("이미 신고했습니다.")

    review.report_count += 1
    if review.report_count >= AUTO_HIDE_THRESHOLD:
        review.status = "HIDDEN_BY_REPORT"
    db.session.commit()
```

---

## 5. Controller

```python
# app/api/reviews.py
from flask import Blueprint, request
from flask_jwt_extended import jwt_required, get_jwt_identity
from marshmallow import Schema, fields, validate

from app.services import review_service
from app.responses import ok

reviews_bp = Blueprint("reviews", __name__, url_prefix="/api/v1")


class CreateReviewSchema(Schema):
    orderItemId = fields.String(required=True, load_default=None, data_key="orderItemId")
    rating = fields.Integer(required=True, validate=validate.Range(min=1, max=5))
    content = fields.String(required=True, validate=validate.Length(min=1, max=2000))
    imageUrls = fields.List(
        fields.URL(),
        load_default=[],
        validate=validate.Length(max=5),
        data_key="imageUrls",
    )


@reviews_bp.post("/products/<product_id>/reviews")
@jwt_required()
def create(product_id: str):
    payload = CreateReviewSchema().load(request.get_json())
    user_id = int(get_jwt_identity())
    review = review_service.create_review(
        db, user_id, product_id, payload["orderItemId"],
        payload["rating"], payload["content"], payload.get("imageUrls"),
    )
    return ok({"reviewId": review.id}, message="리뷰 작성 완료", status=201)


@reviews_bp.post("/reviews/<int:review_id>/helpful")
@jwt_required()
def helpful(review_id: int):
    active = review_service.toggle_helpful(db, review_id, int(get_jwt_identity()))
    return ok({"active": active}, message="도움돼요 표시" if active else "취소")


@reviews_bp.post("/reviews/<int:review_id>/report")
@jwt_required()
def report(review_id: int):
    payload = request.get_json() or {}
    review_service.report_review(
        db, review_id, int(get_jwt_identity()),
        payload.get("reason", "OTHER"), payload.get("detail"),
    )
    return ok(message="신고 접수")
```

---

## 8. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/review#7 함정 모음|§7]] 의 10 함정 그대로)

### Python/Flask 한정

### 함정 11 — `IntegrityError` rollback 누락
`commit()` 실패 후 `session.rollback()` 안 호출하면 다음 query 에 `PendingRollbackError`. **`except IntegrityError: db.session.rollback()`** 강제.

### 함정 12 — `marshmallow.fields.URL()` 가 file:// 도 허용
`schemes=["http", "https"]` 명시 권장 — `URL(schemes=["https"])`.

### 함정 13 — Celery task 안 `db.session`
Flask-SQLAlchemy 의 `db.session` 이 thread-local — Celery worker 는 별도 app context 필요. **`with app.app_context():`** 안에서 호출.

---

## 9. 운영 체크리스트

(Java 원본 §8 그대로)

---

## 10. 관련

- [[../../spring/java-spring/api-design/review|↗ Java 원본]]
- [[file-upload-s3]] — 리뷰 이미지
- [[api-design|↑ hub]]
