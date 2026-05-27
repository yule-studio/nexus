---
title: "troubleshooting · LazyInitializationException (JPA / Hibernate)"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [backend, jpa, hibernate, spring, transaction]
home_hub: "60-troubleshooting"
related:
  - "[[backend]]"
  - "[[n-plus-one-query]]"
---

# LazyInitializationException — `could not initialize proxy ... no Session`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 — OSIV / DTO 매핑 / fetch join 정리 |

## 증상

```
org.hibernate.LazyInitializationException:
  could not initialize proxy [com.example.Author#42] - no Session
    at org.hibernate.proxy.AbstractLazyInitializer.initialize(AbstractLazyInitializer.java:170)
    ...
    at com.example.PostDto.<init>(PostDto.java:12)
```

- 같은 코드가 dev 에서는 통과, prod 에서만 발생
- Spring Boot 2.x → 3.x 업그레이드 후 발생 시작
- 또는 `@RestController` 가 entity 를 직접 반환할 때 Jackson serialization 도중

## 가설

- 트랜잭션이 너무 일찍 끝났나 → 트랜잭션 boundary 검토했더니 service 메서드 안
- Session 이 닫혔나 → entity 가 service 밖으로 detached 상태로 흘러나옴이 사실

## 원인 (root cause)

Hibernate lazy proxy 는 `Session` 이 살아있을 때만 추가 SELECT 가능. 트랜잭션이 끝나면 Session close → proxy 접근 시 예외.

```java
@Service
public class PostService {
    @Transactional(readOnly = true)
    public Post find(Long id) {
        return postRepo.findById(id).orElseThrow();  // ← Session open
    }
    // ← 메서드 종료 시 Session close
}

@RestController
public class PostController {
    public PostDto get(@PathVariable Long id) {
        Post post = service.find(id);                // detached state
        return new PostDto(post.getAuthor().getName());  // ← LazyInit 예외
    }
}
```

Spring Boot 2.x 까지 기본 `spring.jpa.open-in-view = true` (OSIV) → controller render 단계까지 Session 살려둠으로 가려져 있었음. 3.x 부터도 default true 지만 warn 로그.

## 해결

### 즉시 (workaround) — 운영 hotfix 만

`spring.jpa.open-in-view = true` 명시. **권장 안 함** — 트랜잭션 경계가 흐려지고 OSIV 가 N+1 / connection 누수 가림.

### 근본 (3가지)

**(1) DTO 매핑을 트랜잭션 안에서**

```java
@Service
public class PostService {
    @Transactional(readOnly = true)
    public PostDto find(Long id) {  // ← entity 가 아니라 DTO 반환
        Post post = postRepo.findById(id).orElseThrow();
        return new PostDto(post.getId(), post.getTitle(), post.getAuthor().getName());
    }
}
```

원칙: **entity 는 트랜잭션 밖으로 흘리지 않는다**. controller / external API 는 DTO 만.

**(2) Repository 단계에서 fetch join / `@EntityGraph`**

```java
public interface PostRepository extends JpaRepository<Post, Long> {
    @EntityGraph(attributePaths = {"author"})
    Optional<Post> findWithAuthorById(Long id);
}
```

DTO 매핑이 controller 에서 일어나도 author 가 이미 초기화돼 안전.

**(3) JPQL constructor projection** — entity 우회

```java
@Query("SELECT new com.example.PostDto(p.id, p.title, a.name) FROM Post p JOIN p.author a WHERE p.id = :id")
Optional<PostDto> findDtoById(@Param("id") Long id);
```

가장 성능 좋음 (필요 컬럼만). 다만 DTO 시그니처 변경 시 query 수정 필요.

### OSIV 끄기

`application.yml`:
```yaml
spring:
  jpa:
    open-in-view: false
```

→ controller / view 에서 lazy 접근 시 즉시 예외. 강제로 (1)-(3) 패턴 채택.

## 재발 방지

- **OSIV `false`** 가 default — 새 프로젝트는 false 로 시작
- **Architecture rule**: `@RestController` 메서드 시그니처에 `@Entity` 클래스 등장 금지 — ArchUnit / spring-modulith / 코드 리뷰 룰
  ```java
  @Test
  void controllersShouldNotExposeEntities() {
      noClasses()
        .that().resideInAPackage("..controller..")
        .should().haveMethodsReturningType(JavaClasses.from("..entity.."));
  }
  ```
- 회귀 테스트: `@DataJpaTest` 가 아닌 `@WebMvcTest` 로 controller 호출 → MockMvc 응답이 정상 JSON 인지 검증

## 관련

- [[backend]]
- [[n-plus-one-query]] — 같은 lazy 원인, 다른 증상
- [[jackson-lazy-proxy]] — Jackson 이 proxy 직렬화 시도하는 경우
- [[../../20-areas/backend/spring/spring|20-areas/spring]]
