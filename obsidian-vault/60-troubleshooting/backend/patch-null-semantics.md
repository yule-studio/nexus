---
title: "troubleshooting · PATCH 의 null 처리 — 삭제 / 무변경 / 빈문자열"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [backend, spring, jackson, jsr303, patch, api-design, json]
home_hub: "60-troubleshooting"
related:
  - "[[backend]]"
  - "[[npe-deep-chain]]"
---

# PATCH null 처리 — 삭제로 받을지 / 무변경으로 받을지

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 — JsonNullable / Optional / Jackson 동작 / 정책 |

## 증상

`PATCH /users/1` 에 다음 body 가 오면:

```json
// case A — name 필드 자체 없음
{"email": "new@x.com"}

// case B — name 이 null
{"email": "new@x.com", "name": null}

// case C — name 이 빈 문자열
{"email": "new@x.com", "name": ""}
```

각각의 의도:
- **A**: name 은 변경 안 함 (touch X)
- **B**: name 을 null 로 set (= 삭제 / 초기화)
- **C**: name 을 빈 문자열로 set

하지만 일반 DTO 로 받으면 A 와 B 가 둘 다 `name == null` 로 들어옴 → 구분 불가.

```java
public record UserPatchRequest(String email, String name) {}

// case A: name=null  ← 변경 X 의도
// case B: name=null  ← null set 의도
// → 구분 불가 → 잘못 처리하면 의도치 않은 데이터 삭제
```

## 가설

- field 가 null 이면 그냥 무시? → 사용자가 진짜 null 로 set 하고 싶을 때 못함
- field 가 null 이면 null 로 set? → "이 필드 안 건드림" 이 표현 안 됨 → 다른 변경 시도 시 의도 안 한 삭제
- 실제: JSON null vs missing 을 type system level 로 구분해야 함

## 원인 (root cause)

JSON spec 은 `null` 과 missing 을 구분하는데, Java 의 reference type 은 두 경우 모두 `null` 로 매핑됨.

```
JSON          Java 변수
{}            null
{"x": null}   null     ← 같음!
```

Jackson 의 기본 `@JsonProperty` 가 이 정보를 보존 안 함. 별도 wrapper type 필요.

## 해결 (4가지 패턴)

### (1) `JsonNullable<T>` (openapi-tools) — 가장 권장

```xml
<dependency>
  <groupId>org.openapitools</groupId>
  <artifactId>jackson-databind-nullable</artifactId>
  <version>0.2.6</version>
</dependency>
```

```java
import org.openapitools.jackson.nullable.JsonNullable;

public class UserPatchRequest {
    private JsonNullable<String> email = JsonNullable.undefined();
    private JsonNullable<String> name = JsonNullable.undefined();
    // getter / setter
}
```

3-state 가능:
```
undefined  ← field 자체 없음 (case A)
present(null)  ← null 명시 (case B)
present("foo") ← 값 있음
```

Jackson module 등록:
```java
@Bean
public Module jsonNullableModule() {
    return new JsonNullableModule();
}
```

Service 처리:
```java
@Transactional
public void update(Long userId, UserPatchRequest req) {
    User u = userRepo.findById(userId).orElseThrow();

    // email
    if (req.getEmail().isPresent()) {     // case B/C
        u.setEmail(req.getEmail().get());  // null 도 받음 (case B)
    }   // case A → 건드리지 않음

    // name
    if (req.getName().isPresent()) {
        u.setName(req.getName().get());
    }

    userRepo.save(u);
}
```

검증도 가능 (Bean Validation):
```java
public class UserPatchRequest {
    @Email
    @NotNull(message = "email cannot be null if provided")
    private JsonNullable<@Email String> email = JsonNullable.undefined();

    // name 은 null OK (삭제 의미)
    private JsonNullable<@Size(max = 50) String> name = JsonNullable.undefined();
}
```

→ OpenAPI generator 의 default 도 이 패턴. spring 진영 표준에 가까움.

### (2) `Optional<T>` (Jackson 처리 모듈) — 표준 라이브러리

```java
import java.util.Optional;

public class UserPatchRequest {
    private Optional<String> email = Optional.empty();
    private Optional<String> name = Optional.empty();
}
```

하지만 `Optional` 은 null vs absent 구분 못함 — `Optional.empty()` 가 case A/B 둘 다. **추천 안 함** (Jackson 도 일부 어색).

### (3) Set<String> updatedFields — 직접 추적

```java
public class UserPatchRequest {
    private String email;
    private String name;

    private Set<String> _fieldsSet = new HashSet<>();

    @JsonSetter("email")
    public void setEmail(String email) {
        this.email = email;
        _fieldsSet.add("email");
    }

    @JsonSetter("name")
    public void setName(String name) {
        this.name = name;
        _fieldsSet.add("name");
    }

    public boolean has(String field) { return _fieldsSet.contains(field); }
}

// Service
if (req.has("email")) u.setEmail(req.getEmail());
if (req.has("name"))  u.setName(req.getName());
```

원시 + 단순. boilerplate 많음. 작은 DTO 만 OK.

### (4) JSON Patch (RFC 6902) — 표준 spec

```json
PATCH /users/1
Content-Type: application/json-patch+json

[
  { "op": "replace", "path": "/email", "value": "new@x.com" },
  { "op": "remove",  "path": "/name" }
]
```

```xml
<dependency>
  <groupId>com.github.java-json-tools</groupId>
  <artifactId>json-patch</artifactId>
  <version>1.13</version>
</dependency>
```

```java
@PatchMapping(path = "/users/{id}", consumes = "application/json-patch+json")
public UserResponse patch(@PathVariable Long id, @RequestBody JsonPatch patch)
        throws JsonPatchException, JsonProcessingException {
    User u = userRepo.findById(id).orElseThrow();
    JsonNode patched = patch.apply(objectMapper.valueToTree(u));
    User updated = objectMapper.treeToValue(patched, User.class);
    return UserResponse.from(userRepo.save(updated));
}
```

장점: 표준. 복잡한 부분 update (배열 item 추가 / 제거) 지원.
단점: client 가 복잡. Bean Validation 적용 어려움. 일반 REST API 에는 over-engineering.

JSON Merge Patch (RFC 7396) — 더 단순:
```json
{"name": null, "email": "new@x.com"}   // name 은 삭제, email 은 set
```

→ null = 삭제 가 spec. 단 "값을 null 로 set" 표현 불가.

## 정책 결정 표

| 시나리오 | 권장 |
| --- | --- |
| nullable field 가 거의 없음 (모두 required) | 단순 DTO + null = 무변경 |
| nullable field 가 있고 null set 도 의미 있음 | **(1) JsonNullable** |
| 외부 client / OpenAPI spec | (1) JsonNullable (generator default) |
| 표준 / spec 준수 / 복잡한 batch update | (4) JSON Patch |
| 작은 DTO (3개 필드 이하) | (3) updatedFields 직접 |

## 흔한 잘못된 패턴

**(1) `@DynamicUpdate` + `null = 무변경` 가정**

```java
@Entity
@DynamicUpdate
public class User {
    String email;
    String name;
}

// Service
public void update(Long id, UserPatchRequest req) {
    User u = userRepo.findById(id).orElseThrow();
    if (req.email() != null) u.setEmail(req.email());
    if (req.name() != null)  u.setName(req.name());
}
```

→ 사용자가 진짜 name 을 null 로 비우고 싶을 때 못함. 또는 빈 문자열 `""` 로 회피 → DB 의 NOT NULL 위반.

**(2) Service 에서 null check 분기**

```java
// 의도 모호한 코드
if (req.email() != null && !req.email().isEmpty()) {
    u.setEmail(req.email());
}
```

빈 문자열을 의도된 set 으로 받아야 할 때 차단됨. 의도 분리 안 됨.

**(3) `@JsonInclude(Include.NON_NULL)` 응답에 적용 — 요청에 무관**

```java
@JsonInclude(Include.NON_NULL)
public class UserResponse { ... }
```

→ 응답에서 null 필드 숨김. 요청 처리와 무관 — 혼동하지 말 것.

## Validation 함정

`@Valid` 가 JsonNullable 안 풀어줌 → 직접 풀어야 함:

```java
public class UserPatchRequest {
    private JsonNullable<@Email String> email = JsonNullable.undefined();
}
```

또는:
```java
@AssertTrue(message = "email must be valid if provided")
public boolean isEmailValid() {
    if (!email.isPresent() || email.get() == null) return true;
    return email.get().matches("...");
}
```

## 회귀 테스트

```java
@Test
void missingFieldShouldNotChange() throws Exception {
    var user = userRepo.save(User.of("a@x.com", "Alice"));

    mockMvc.perform(patch("/users/{id}", user.getId())
        .contentType("application/json")
        .content("""{"email": "b@x.com"}"""))   // name 없음
        .andExpect(status().isOk());

    var refreshed = userRepo.findById(user.getId()).orElseThrow();
    assertThat(refreshed.getEmail()).isEqualTo("b@x.com");
    assertThat(refreshed.getName()).isEqualTo("Alice");   // ← 그대로
}

@Test
void explicitNullShouldClear() throws Exception {
    var user = userRepo.save(User.of("a@x.com", "Alice"));

    mockMvc.perform(patch("/users/{id}", user.getId())
        .contentType("application/json")
        .content("""{"name": null}"""))
        .andExpect(status().isOk());

    var refreshed = userRepo.findById(user.getId()).orElseThrow();
    assertThat(refreshed.getName()).isNull();   // ← 삭제됨
}

@Test
void emptyStringShouldBeAccepted() throws Exception {
    var user = userRepo.save(User.of("a@x.com", "Alice"));

    mockMvc.perform(patch("/users/{id}", user.getId())
        .contentType("application/json")
        .content("""{"name": ""}"""))
        .andExpect(status().isOk());

    var refreshed = userRepo.findById(user.getId()).orElseThrow();
    assertThat(refreshed.getName()).isEmpty();   // 빈 문자열 (null 아님)
}
```

## API 문서화 (OpenAPI)

```yaml
components:
  schemas:
    UserPatchRequest:
      type: object
      properties:
        email:
          type: string
          format: email
          nullable: false       # 누락 = 무변경, null = 거절
        name:
          type: string
          nullable: true        # 누락 = 무변경, null = 삭제
```

OpenAPI generator (codegen) 가 `nullable: true` 면 `JsonNullable<String>` 생성. spec ↔ code 일관.

## 재발 방지

- **결정 룰**: 새 PATCH endpoint 추가 시 각 nullable field 의 null 의미 명시 (삭제 / 거절 / 변경 X)
- **DTO 규약**:
  - PATCH 요청은 `JsonNullable<T>` 우선
  - POST/PUT 은 단순 type (PUT 은 전체 replace 이므로 null 도 명시적)
- **API 문서 (OpenAPI)** 에 `nullable: true/false` 명시 — schema 가 SSoT
- **회귀 테스트 세트**: 모든 PATCH endpoint 에 (missing / null / empty / valid) 4 케이스 검증

## 관련

- [[backend]]
- [[npe-deep-chain]] — null 처리 일반
- [[../../20-areas/backend/spring/spring|20-areas/spring]]
- RFC 6902 (JSON Patch) / RFC 7396 (JSON Merge Patch)
- 책: REST API Design Rulebook / Building Microservices 2nd Ed
