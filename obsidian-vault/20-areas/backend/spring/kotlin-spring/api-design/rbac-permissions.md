---
title: "RBAC + Permissions — Role-based / Hierarchical / @PreAuthorize (Kotlin)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T13:10:00+09:00
tags:
  - backend
  - kotlin-spring
  - api-design
  - recipe
  - auth
  - rbac
  - authorization
---

# RBAC + Permissions — Role-based / Hierarchical / @PreAuthorize (Kotlin)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + enum 멤버 함수 + 함정 |

**[[api-design|↑ api-design hub]]** · **[[../../java-spring/api-design/rbac-permissions|↗ Java 원본]]**

> 📐 공통: [[../common/response-envelope]] · [[../common/security-config]].

---

## 1. 무엇을 만드는가 / 1.1 RBAC vs ABAC vs ACL / 7. 운영·감사 / 8. 함정

(Java 원본 §1, §7, §8 와 동일)

---

## 2. 도메인

### 2.1 Role / Permission

```kotlin
// domain/auth/Role.kt
enum class Role {
    USER, SELLER, ADMIN, MASTER;

    fun asAuthority(): String = "ROLE_$name"
}

// domain/auth/Permission.kt
enum class Permission {
    PRODUCT_READ, PRODUCT_WRITE, PRODUCT_DELETE,
    ORDER_READ_OWN, ORDER_READ_ANY, ORDER_REFUND,
    USER_READ, USER_WRITE, USER_DELETE,
    KEY_ROTATE, AUDIT_READ;

    fun asAuthority(): String = "PERM_$name"
}
```

### 2.2 Role → Permission 매핑

```kotlin
object RolePermissions {
    private val MAP: Map<Role, Set<Permission>> = mapOf(
        Role.USER to setOf(
            Permission.PRODUCT_READ,
            Permission.ORDER_READ_OWN,
        ),
        Role.SELLER to setOf(
            Permission.PRODUCT_READ, Permission.PRODUCT_WRITE,
            Permission.ORDER_READ_OWN,
        ),
        Role.ADMIN to setOf(
            Permission.PRODUCT_READ, Permission.PRODUCT_WRITE, Permission.PRODUCT_DELETE,
            Permission.ORDER_READ_ANY, Permission.ORDER_REFUND,
            Permission.USER_READ, Permission.USER_WRITE,
            Permission.AUDIT_READ,
        ),
        Role.MASTER to Permission.entries.toSet(),
    )

    fun of(role: Role): Set<Permission> = MAP[role].orEmpty()

    fun authorities(role: Role): Set<String> =
        buildSet {
            add(role.asAuthority())
            of(role).mapTo(this) { it.asAuthority() }
        }
}
```

> **Kotlin idiom 포인트**:
> - `object RolePermissions` — Java `final class` + private constructor.
> - `Permission.entries.toSet()` — Kotlin 1.9+ 의 enum entries (Java 의 `values()` 대신).
> - `buildSet { add(...); mapTo(this) { } }` — mutable 빌더 한 expression.

### 2.3 hierarchy (옵션)

```kotlin
@Bean
fun roleHierarchy(): RoleHierarchy = RoleHierarchyImpl.fromHierarchy(
    """
    ROLE_MASTER > ROLE_ADMIN
    ROLE_ADMIN > ROLE_SELLER
    ROLE_SELLER > ROLE_USER
    """.trimIndent(),
)

@Bean
fun methodSecurityExpressionHandler(hierarchy: RoleHierarchy): DefaultMethodSecurityExpressionHandler =
    DefaultMethodSecurityExpressionHandler().apply { setRoleHierarchy(hierarchy) }
```

---

## 3. JWT 발급 시 authorities 포함

```kotlin
fun generateAccessToken(email: String, userId: Long, role: Role, sessionId: String): String {
    val authorities = RolePermissions.authorities(role)
    return Jwts.builder()
        .issuer(issuer)
        .subject(email)
        .id(UUID.randomUUID().toString())
        .claim(CLAIM_TYPE, "ACCESS")
        .claim(CLAIM_ID, userId)
        .claim(CLAIM_ROLE, role.asAuthority())
        .claim("authorities", authorities.toList())          // JWT 는 List 직렬화 권장
        .claim(CLAIM_SID, sessionId)
        .issuedAt(Date.from(Instant.now()))
        .expiration(Date.from(Instant.now().plus(accessTtl)))
        .signWith(key, Jwts.SIG.HS256)
        .compact()
}
```

`JwtAuthenticationFilter` 에서:
```kotlin
@Suppress("UNCHECKED_CAST")
val rawAuthorities = claims["authorities"] as? List<String> ?: emptyList()
val grantedAuthorities = rawAuthorities.map(::SimpleGrantedAuthority)
```

> **함정**: authorities 가 너무 많으면 JWT 가 커짐 (cookie 4KB limit 위험). 50개 이상이면 server lookup 으로 전환.

---

## 4. Controller / Service 적용

### 4.1 Endpoint 단 — `@PreAuthorize`

```kotlin
@RestController
@RequestMapping("/api/v1/seller/products")
class SellerProductController(/* ... */) {

    @PreAuthorize("hasRole('SELLER')")
    @PostMapping
    fun create(@Valid @RequestBody req: CreateProductRequest, auth: Authentication): ResponseEntity<*> {
        TODO()
    }

    @PreAuthorize("hasAuthority('PERM_PRODUCT_DELETE')")
    @DeleteMapping("/{id}")
    fun delete(@PathVariable id: String): ResponseEntity<*> {
        TODO()
    }

    @PreAuthorize("hasRole('SELLER') and @productSecurity.canEdit(#id, principal)")
    @PatchMapping("/{id}")
    fun update(
        @PathVariable id: String,
        @Valid @RequestBody req: UpdateProductRequest,
        auth: Authentication,
    ): ResponseEntity<*> {
        TODO()
    }
}
```

### 4.2 ownership — custom SpEL bean

```kotlin
@Component("productSecurity")
class ProductSecurity(
    private val products: ProductRepository,
) {

    fun canEdit(productId: String, principal: Any?): Boolean {
        val user = principal as? User ?: return false
        if (user.role in setOf(Role.ADMIN, Role.MASTER)) return true
        return products.findById(ProductId(productId))?.isOwnedBy(user.id) ?: false
    }

    fun canDelete(productId: String, principal: Any?): Boolean = canEdit(productId, principal)
}
```

### 4.3 Service 단 — 도메인 검증 (이중)

```kotlin
@Service
class UpdateProductUseCase(
    private val products: ProductRepository,
) {
    @Transactional
    fun handle(id: ProductId, currentUser: UserId, currentRole: Role, /* ... */) {
        val product = products.findById(id) ?: throw BusinessException(ResponseCode.NOT_FOUND)

        if (!product.isOwnedBy(currentUser) && currentRole !in setOf(Role.ADMIN, Role.MASTER)) {
            throw BusinessException(ResponseCode.FORBIDDEN, "본인의 상품만 수정 가능합니다.")
        }
        // ...
    }
}
```

> **규칙**: Controller `@PreAuthorize` + Service 도메인 체크 **이중**.

---

## 5. Method security 활성 / 6. Permission 동적 변경

(Java 원본 §5, §6 와 동일)

---

## 9. 함정 모음

(원본 [[../../java-spring/api-design/rbac-permissions#8 함정 모음|§8]] 의 10가지 함정 그대로)

### Kotlin 한정 추가 함정

### 함정 11 — `enum.values()` → `entries`
Kotlin 1.9+ 에선 `Permission.values()` deprecated, `Permission.entries` 권장 (allocation 없음).

### 함정 12 — `@PreAuthorize` SpEL 의 `#id` 와 Kotlin parameter
SpEL 의 `#id` 는 Java parameter name. Kotlin 도 동일하지만 컴파일 시 parameter name 보존 옵션 (`-java-parameters`) 또는 `kotlin("plugin.spring")` 의 기본 active 확인.

### 함정 13 — `as?` cast 의 fallback
`principal as? User ?: return false` — Java `instanceof + cast` 한 줄. principal 이 String (Spring Security anonymous) 일 때 false 반환.

---

## 10. 운영 체크리스트

(Java 원본 §9 그대로 + Kotlin)

- [ ] `Permission.entries` (1.9+) 사용
- [ ] `kotlin("plugin.spring")` 으로 method security proxy 활성
- [ ] `-java-parameters` 컴파일 옵션 확인

---

## 11. 관련

- [[../../java-spring/api-design/rbac-permissions|↗ Java 원본]]
- [[../common/security-config]] — `@EnableMethodSecurity`
- [[two-factor-auth]]
- [[api-design|↑ api-design hub]]
