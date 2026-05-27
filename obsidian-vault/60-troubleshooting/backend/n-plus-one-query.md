---
title: "troubleshooting · N+1 query (JPA / Hibernate)"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [backend, jpa, hibernate, spring, performance, n+1]
home_hub: "60-troubleshooting"
related:
  - "[[backend]]"
  - "[[../database/database|database 트러블]]"
---

# N+1 query — list API 가 1 → 100+ SELECT 로 폭증

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 — JPA fetch join / @EntityGraph / batch size 정리 |

## 증상

- list API 응답 시간이 데이터 증가에 따라 선형 폭증 (100ms → 5s)
- 로그에 같은 패턴 쿼리 100+ 회 반복:
  ```sql
  -- 최초 1 쿼리
  SELECT * FROM post LIMIT 20;
  -- 그 다음 N 쿼리 (post 마다 1번)
  SELECT * FROM author WHERE id = ?;
  SELECT * FROM author WHERE id = ?;
  ... (N회)
  ```
- DataDog / New Relic 가 같은 endpoint 의 db.query count 급증 알람

## 가설

- 처음에 의심: DB index 미사용 → `EXPLAIN` 확인 했으나 author table 의 PK lookup 은 정상
- 다음 의심: connection pool 부족 → HikariCP active=2/max=20 으로 여유 있음
- 실제: lazy 연관 entity 가 list iteration 중 1건씩 lazy load

## 원인 (root cause)

JPA 의 `@ManyToOne` / `@OneToMany` 는 기본 `FetchType.LAZY`. list 를 받아 author 를 접근하는 순간 proxy 가 SELECT 발행.

```java
// Post.java
@Entity
public class Post {
    @ManyToOne(fetch = FetchType.LAZY)  // ← 기본값
    private Author author;
}

// PostService.java
List<Post> posts = postRepository.findAll();  // 1 query
return posts.stream()
    .map(p -> new PostDto(p.getTitle(), p.getAuthor().getName()))  // ← p.getAuthor() 마다 N query
    .toList();
```

JPA 1차 캐시는 트랜잭션 scope. author 가 모두 다른 id 면 cache hit 없음 → N 회 발행.

## 해결

### 즉시 (workaround)

— 없음. 운영에서는 코드 수정 + 재배포가 가장 빠른 길.

### 근본 (3가지 패턴)

**(1) JPQL `JOIN FETCH`** — 단일 collection 까지

```java
@Query("SELECT p FROM Post p JOIN FETCH p.author")
List<Post> findAllWithAuthor();
```

주의: collection (`@OneToMany`) 을 `JOIN FETCH` 하면 cartesian product → `distinct` 또는 Set 받기. 2개 이상 collection 동시 fetch 는 `MultipleBagFetchException`.

**(2) `@EntityGraph`** — declarative, repository method 에 부착

```java
public interface PostRepository extends JpaRepository<Post, Long> {
    @EntityGraph(attributePaths = {"author", "tags"})
    List<Post> findAll();
}
```

장점: 같은 entity 에 여러 graph 정의 가능 (`@NamedEntityGraph`). pagination 과 잘 어울림 (in-memory pagination 회피).

**(3) `@BatchSize`** — N+1 → ceil(N/batch) +1

```java
@Entity
public class Post {
    @ManyToOne(fetch = FetchType.LAZY)
    @BatchSize(size = 100)  // 또는 hibernate.default_batch_fetch_size = 100 (global)
    private Author author;
}
```

100 post → 1 (post) + 1 (author IN (...)) = 2 query. fetch join 보다 안전 (cartesian X, pagination O).

### Pagination + fetch join 함정

`Pageable` + `JOIN FETCH collection` 조합은 Hibernate 가 in-memory pagination 경고:

```
firstResult/maxResults specified with collection fetch; applying in memory
```

→ 전체 row 로딩 후 메모리에서 자르기. 대용량에 치명적.

대안:
- 2-step query: id 만 page → id IN (...) + fetch join
- `@EntityGraph` + `Pageable` (single-valued association 만 그래프, collection 은 `@BatchSize`)

## 재발 방지

- **`spring.jpa.properties.hibernate.default_batch_fetch_size = 100`** — global default 깔아두기 (N+1 → 2)
- **테스트에서 `@DataJpaTest` + `Statistics`** 로 query count assert:
  ```java
  @Autowired EntityManagerFactory emf;

  @Test
  void postListShouldNotTriggerNPlusOne() {
      var stats = emf.unwrap(SessionFactory.class).getStatistics();
      stats.setStatisticsEnabled(true);
      stats.clear();
      service.findAll();
      assertThat(stats.getPrepareStatementCount()).isLessThanOrEqualTo(2);
  }
  ```
- **`p6spy` / `datasource-proxy`** 로 dev 환경 SQL log + count 표면화
- PR 검토: list / detail API 추가 시 fetch strategy 명시 요구

## 관련

- [[backend]]
- [[lazy-initialization-exception]] — 같은 lazy 원인의 다른 증상
- [[../database/database|database 트러블]] — DB 측 slow query 분석
- 책: [[../../30-resources/books/backend|effective-jpa / 자바 ORM 표준 프로그래밍]]
