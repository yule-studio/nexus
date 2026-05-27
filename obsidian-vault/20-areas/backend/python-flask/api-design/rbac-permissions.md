---
title: "RBAC + Permissions — Flask"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T17:20:00+09:00
tags: [backend, python-flask, api-design, recipe, auth, rbac]
---

# RBAC + Permissions — Flask

**[[api-design|↑ hub]]** · **[[../../spring/java-spring/api-design/rbac-permissions|↗ Java 원본]]**

> [[../common/security-config]] 의 `@role_required` decorator 가 본 레시피 핵심.

---

## 1. 무엇을 만드는가 / 7. 운영·감사 / 8. 함정

(Java 원본 [[../../spring/java-spring/api-design/rbac-permissions#1|§1, §7, §8]] 와 동일)

---

## 2. 도메인 — Role / Permission enum

```python
# app/domain/auth.py
from enum import Enum, auto


class Role(Enum):
    USER = auto()
    SELLER = auto()
    ADMIN = auto()
    MASTER = auto()

    @property
    def authority(self) -> str:
        return f"ROLE_{self.name}"


class Permission(Enum):
    PRODUCT_READ = auto()
    PRODUCT_WRITE = auto()
    PRODUCT_DELETE = auto()
    ORDER_READ_OWN = auto()
    ORDER_READ_ANY = auto()
    USER_READ = auto()
    USER_WRITE = auto()
    AUDIT_READ = auto()
    KEY_ROTATE = auto()

    @property
    def authority(self) -> str:
        return f"PERM_{self.name}"


ROLE_PERMISSIONS: dict[Role, set[Permission]] = {
    Role.USER: {Permission.PRODUCT_READ, Permission.ORDER_READ_OWN},
    Role.SELLER: {Permission.PRODUCT_READ, Permission.PRODUCT_WRITE, Permission.ORDER_READ_OWN},
    Role.ADMIN: {
        Permission.PRODUCT_READ, Permission.PRODUCT_WRITE, Permission.PRODUCT_DELETE,
        Permission.ORDER_READ_ANY, Permission.USER_READ, Permission.USER_WRITE,
        Permission.AUDIT_READ,
    },
    Role.MASTER: set(Permission),
}


def authorities_for(role: Role) -> set[str]:
    return {role.authority, *(p.authority for p in ROLE_PERMISSIONS.get(role, set()))}
```

---

## 3. JWT 에 role / authorities

[[../common/security-config#5 JWT token 발급]] 참고. `additional_claims = {"role": user.role.name, ...}`.

---

## 4. Endpoint 보호 — `@role_required` decorator

[[../common/security-config#6 권한]] 의 `@role_required("ADMIN", "MASTER")` 그대로 사용.

```python
from app.common.auth import role_required


@admin_bp.delete("/products/<product_id>")
@role_required("ADMIN", "MASTER")
def delete_product(product_id: str):
    # ...
    return ok("삭제")
```

---

## 5. ownership 검증

```python
from app.common.auth import owns_or_admin


def _is_product_owner(claims: dict, product_id: str) -> bool:
    user_id = claims.get("id")
    product = product_repo.find_by_id(product_id)
    return product is not None and product.seller_id == user_id


@seller_bp.patch("/products/<resource_id>")
@owns_or_admin(_is_product_owner)
def update_product(resource_id: str):
    # ...
    return ok("수정")
```

---

## 6. Service 단 도메인 검증 (이중)

```python
def update_product(product_id: str, current_user_id: int, current_role: Role, payload: dict) -> Product:
    product = product_repo.find_by_id(product_id)
    if product is None:
        raise BusinessException(ResponseCode.NOT_FOUND)

    if not product.is_owned_by(current_user_id) and current_role not in (Role.ADMIN, Role.MASTER):
        raise BusinessException(ResponseCode.FORBIDDEN, "본인의 상품만 수정 가능합니다.")

    # ...
    return product
```

→ Endpoint decorator + Service 양쪽 검증.

---

## 9. 함정 모음

(Java 원본 [[../../spring/java-spring/api-design/rbac-permissions#8 함정 모음|§8]] 의 10 함정 그대로)

### Python/Flask 한정

### 함정 11 — `Role.USER == "USER"` (str 비교)
`Role` 이 Enum 이라 str 과 직접 비교 시 항상 False. **`role.name == "USER"`** 또는 `role is Role.USER`.

### 함정 12 — JWT claim 의 `role` 가 str 인 채로 enum 변환 안 됨
`Role[claims["role"]]` — KeyError 가능. `Role.__members__.get(claims["role"])` 또는 try/except.

### 함정 13 — `@role_required` 가 `@jwt_required` 보다 outer
decorator 순서 — `@role_required` 안에 `@jwt_required` 가 이미 있음 (위 §4 참고). 또 붙이지 말 것.

---

## 10. 운영 체크리스트

(Java 원본 §9 그대로 + Python: Role enum / claims 매핑 type-safe)

---

## 11. 관련

- [[../common/security-config]] — `@role_required` 본거지
- [[../../spring/java-spring/api-design/rbac-permissions|↗ Java 원본]]
- [[api-design|↑ hub]]
