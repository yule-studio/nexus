---
title: "RBAC + Permissions — FastAPI"
kind: knowledge
status: current
created_at: 2026-05-26T23:50:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, python-fastapi, api-design, recipe, auth, rbac]
---

# RBAC + Permissions — FastAPI

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/rbac-permissions|↗ Java 원본]]** · **[[../../python-flask/api-design/rbac-permissions|↗ Flask 변형]]**

> [[../common/security-config#5 권한]] 의 `require_role` factory 가 본 레시피 핵심.

---

## 1. 무엇을 / 7. 감사 / 8. 함정

(Java 원본 [[../../spring/java-spring/api-design/rbac-permissions#1|§1, §7, §8]] 와 동일)

---

## 2. Role / Permission enum

(Flask 변형 [[../../python-flask/api-design/rbac-permissions#2 도메인|§2]] 의 `Role` / `Permission` enum + `ROLE_PERMISSIONS` 매핑 동일 — Python enum 표준)

---

## 3. JWT 에 role / authorities

[[../common/security-config#5 JWT token 발급]] 참고. `additional={"role": user.role.name, ...}`.

---

## 4. Endpoint 보호 — Depends(require_role)

```python
from typing import Annotated
from fastapi import APIRouter, Depends

from app.deps import require_role
from app.domain.user import User

router = APIRouter(prefix="/admin", tags=["admin"])


@router.delete("/products/{product_id}")
async def delete_product(
    product_id: str,
    user: Annotated[User, Depends(require_role("ADMIN", "MASTER"))],
):
    await product_service.delete(product_id)
    return {"deleted": product_id}
```

> **FastAPI idiom**: factory 함수 `require_role(...)` 가 dependency 반환. Java `@PreAuthorize` 와 다르게 type-safe.

---

## 5. ownership 검증 — Depends

```python
# app/deps.py 에
async def owns_or_admin_product(
    product_id: str,
    user: Annotated[User, Depends(get_current_user)],
    db: Annotated[AsyncSession, Depends(get_db)],
) -> User:
    if user.role in (Role.ADMIN, Role.MASTER):
        return user
    product = await db.get(Product, product_id)
    if product is None or product.seller_id != user.id:
        raise ForbiddenException("본인의 상품만 접근 가능합니다.")
    return user


@router.patch("/products/{product_id}")
async def update_product(
    product_id: str,
    payload: UpdateProductRequest,
    user: Annotated[User, Depends(owns_or_admin_product)],
    db: SessionDep,
):
    await product_service.update(db, product_id, payload)
    return {"updated": product_id}
```

> **FastAPI idiom**: path parameter (`product_id`) 가 dependency 함수의 인자에도 자동 주입. 한 번만 선언.

---

## 6. Service 단 도메인 검증

```python
async def update_product(
    db: AsyncSession, product_id: str,
    current_user_id: int, current_role: Role, payload: dict,
) -> Product:
    product = await db.get(Product, product_id)
    if product is None:
        raise NotFoundException("상품")

    if not product.is_owned_by(current_user_id) and current_role not in (Role.ADMIN, Role.MASTER):
        raise ForbiddenException("본인의 상품만 수정 가능합니다.")

    # ...
    return product
```

→ Endpoint Depends + Service 양쪽 검증.

---

## 9. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/rbac-permissions#8 함정 모음|§8]] 의 10 함정 그대로)

### FastAPI 한정

### 함정 11 — `Depends(require_role(...))` 의 closure / cache
FastAPI 가 `Depends(X)` 호출마다 새 X 실행. `require_role("ADMIN")` 가 매 호출마다 새 함수 — `@lru_cache` 로 최적화 가능하지만 보통 불필요.

### 함정 12 — Role enum → JWT 직렬화
`Role.ADMIN` 그대로 JWT claim 에 넣으면 `<Role.ADMIN: 3>` 직렬화. **`role=user.role.name`** 또는 `str(user.role)`.

### 함정 13 — `Annotated[User, Depends(...)]` 위치
function signature 에서 `Depends` 인자가 path / query / body 보다 뒤에 와야 깔끔 (FastAPI 0.95+ 권장). default value 인자는 마지막.

---

## 10. 운영 체크리스트

(Java 원본 §9 + Python: Role enum str 매핑 / require_role factory)

---

## 11. 관련

- [[../common/security-config]] — require_role 본거지
- [[../../spring/java-spring/api-design/rbac-permissions|↗ Java 원본]]
- [[../../python-flask/api-design/rbac-permissions|↗ Flask 변형]]
- [[api-design|↑ hub]]
