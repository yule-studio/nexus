---
title: "troubleshooting · Jackson — failed to lazily initialize (proxy serialization)"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [backend, spring, jackson, jpa, hibernate, serialization]
home_hub: "60-troubleshooting"
related:
  - "[[backend]]"
  - "[[lazy-initialization-exception]]"
---

# Jackson — `failed to lazily initialize a collection` (JPA proxy 직렬화)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 — DTO 분리 / Hibernate5/6Module / @JsonIgnore |

## 증상

```
com.fasterxml.jackson.databind.JsonMappingException:
  failed to lazily initialize a collection of role: Post.comments,
  could not initialize proxy - no Session
    (through reference chain: PostDto["comments"])
    at com.fasterxml.jackson.databind.JsonMappingException.wrapWithPath
```

- `@RestController` 가 entity 를 직접 반환
- 또는 entity 를 포함한 wrapper 객체 반환
- Hibernate proxy 가 serialization 도중 lazy load 시도

## 가설

- DTO 매핑 누락? → controller 에 ResponseEntity<Entity> 가 명백히 보임
- transaction 이 일찍 끝나 lazy 실패? → 맞지만 더 근본 원인은 entity 노출

## 원인 (root cause)

Jackson 은 객체의 모든 getter 를 호출해 JSON 으로 변환. lazy proxy 의 getter 호출 → Session 필요 → 트랜잭션 밖이면 예외.

```java
@GetMapping("/posts/{id}")
public Post get(@PathVariable Long id) {
    return service.find(id);   // entity 반환 — 안티패턴
}

@Entity
public class Post {
    @OneToMany(mappedBy = "post", fetch = LAZY)
    private List<Comment> comments;   // ← Jackson 가 호출 시 lazy 발행
}
```

## 해결

### 즉시 (workaround) — 가능하면 피할 것

**(1) Hibernate5Module / Hibernate6Module 등록**

```xml
<dependency>
  <groupId>com.fasterxml.jackson.datatype</groupId>
  <artifactId>jackson-datatype-hibernate6</artifactId>
</dependency>
```

```java
@Bean
public Module hibernateModule() {
    Hibernate6Module m = new Hibernate6Module();
    m.disable(Hibernate6Module.Feature.USE_TRANSIENT_ANNOTATION);
    return m;
}
```

→ lazy 미초기화 proxy 는 `null` 로 직렬화 (예외 회피).

문제: lazy 가 null 로 보이는데 진짜 데이터가 null 인지 lazy 미로딩인지 구분 불가. **권장 안 함**.

### 근본 (DTO 분리 — 유일한 옳은 답)

```java
// Controller
@GetMapping("/posts/{id}")
public PostResponse get(@PathVariable Long id) {
    return service.findResponse(id);   // ← DTO
}

// Service
@Transactional(readOnly = true)
public PostResponse findResponse(Long id) {
    Post p = repo.findWithDetailsById(id).orElseThrow();
    return PostResponse.from(p);   // entity → DTO (트랜잭션 안에서)
}

// DTO
public record PostResponse(
    Long id,
    String title,
    String authorName,
    List<CommentResponse> comments
) {
    public static PostResponse from(Post p) {
        return new PostResponse(
            p.getId(),
            p.getTitle(),
            p.getAuthor().getName(),
            p.getComments().stream().map(CommentResponse::from).toList()
        );
    }
}
```

**원칙**: `@RestController` → DTO 만. entity 가 method signature 에 등장하면 reject.

### 보조 — DTO 변환 자동화 (MapStruct)

```java
@Mapper(componentModel = "spring")
public interface PostMapper {
    @Mapping(source = "author.name", target = "authorName")
    PostResponse toResponse(Post post);
}

// Controller
return mapper.toResponse(service.find(id));
```

빌드 시 코드 생성 → reflection 없음 → 빠름. 단 컴파일러 plugin 필요.

### entity 측 보조 — 예방

```java
@Entity
@JsonIgnoreProperties({"hibernateLazyInitializer", "handler"})   // proxy 내부 필드 무시
public class Post {

    @OneToMany(mappedBy = "post", fetch = LAZY)
    @JsonIgnore   // ← 또는 항상 무시
    private List<Comment> comments;
}
```

→ entity 직접 반환 시 안전망. 하지만 DTO 분리가 옳음.

### JPQL constructor projection — 가장 빠름

```java
@Query("""
    SELECT new com.example.PostResponse(p.id, p.title, a.name)
    FROM Post p JOIN p.author a
    WHERE p.id = :id
    """)
Optional<PostResponse> findResponseById(@Param("id") Long id);
```

entity 거치지 않음 → lazy 문제 자체 없음.

## OSIV 와의 관계

`spring.jpa.open-in-view = true` (default) 이면 view rendering 단계까지 Session 살림 → Jackson serialization 시점에도 lazy load 가능 → **예외는 안 나지만 N+1 query 폭발**.

```
posts.json
→ Jackson 이 post[0].author.getName() → SELECT FROM author WHERE id=1
→ Jackson 이 post[1].author.getName() → SELECT FROM author WHERE id=2
... (100회)
```

→ OSIV `false` + DTO 가 옳음. [[lazy-initialization-exception]] 참조.

## 진단

```java
// 응답 JSON 의 dollar sign 검사 — proxy 노출 신호
{
  "id": 1,
  "title": "...",
  "comments_$$_hibernate_interceptor": {...}   // ← 실패 신호
}
```

```bash
# Jackson serialization 로그
logging.level.com.fasterxml.jackson=DEBUG

# 응답에 entity 가 포함되었는지 단위 테스트
mockMvc.perform(get("/posts/1"))
  .andExpect(jsonPath("$.comments_$$_hibernate_interceptor").doesNotExist());
```

## 재발 방지

- **ArchUnit 룰**: `@RestController` 메서드 시그니처에 `@Entity` 클래스 등장 금지
  ```java
  @Test
  void controllersShouldNotReturnEntities() {
      noClasses().that().areAnnotatedWith(RestController.class)
        .should().haveMethodsThat().haveRawReturnType(
            classesAnnotatedWith(Entity.class));
  }
  ```
- **OSIV `false`** + 통합 테스트로 lazy 노출 차단
- **DTO 변환 강제**: PR 검토 시 entity 반환 발견 → 무조건 변경 요청
- **테스트**: `@WebMvcTest` 로 JSON 형식 검증 (entity 내부 필드 노출 X)

## 관련

- [[backend]]
- [[lazy-initialization-exception]] — 같은 영역의 다른 증상
- [[n-plus-one-query]] — OSIV 시 동반 발생
- [[npe-deep-chain]] — null 처리와 함께 고민
