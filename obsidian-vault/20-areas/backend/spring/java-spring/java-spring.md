---
title: "Java + Spring — Hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
updated_at: 2026-05-19T01:15:00+09:00
tags: [area, backend, java-spring, hub]
home_hub: spring
related:
  - "[[../spring]]"
  - "[[../../backend]]"
  - "[[api-design/api-design]]"
  - "[[database/database]]"
  - "[[common/response-envelope]]"
  - "[[common/security-config]]"
  - "[[pitfalls/pitfalls]]"
  - "[[../../../computer-science/object-oriented-programming/oop-for-backend]]"
---

# Java + Spring — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 최초 placeholder |
| v.2.0.0 | 2026-05-19 | engineering-agent/tech-lead | placeholder → 본 hub: 영역 분류 / 학습 순서 / api-design 19 레시피 인덱스 / 공통 / DB / pitfalls / api-docs 연결 |
| v.2.1.0 | 2026-05-19 | engineering-agent/tech-lead | observability/ (로그 + 메트릭 모니터링) 추가 — Logback / MDC / ELK + Actuator / Prometheus / Grafana / Discord Alert |
| v.2.2.0 | 2026-05-20 | engineering-agent/tech-lead | bkurs-be 시작 회수 — database/flyway / pitfalls/spring-initializr-version 추가 |

**[[../spring|↑ Spring framework hub]]** · **[[../../backend|↑ backend hub]]**

---

## 1. 영역 정의

본 영역은 **Java + Spring Boot 3** 으로 백엔드 API 를 구현하기 위한 모든 노트의 진입점이다. Spring framework 자체 (버전 / Java/Kotlin 비교 / 모듈 분류) 는 [[../spring|↑ spring hub]] 가 담당. 본 hub 는 **Java 한정 + 실전 구현 / 운영** 에 집중한다.

본 영역이 정의하는 것:
- 6 디렉토리 (api-design / api-docs / common / database / pitfalls) 의 진입점
- 단계별 학습 순서 (입문 → 운영 → 분산)
- api-design 의 19 실전 레시피 인덱스 (auth / CRUD / 검색 / 통신 / 분산 / 운영)
- 공통 (response envelope / security config) / DB (JPA / MyBatis / 공존) / pitfalls 연결
- OOP / DDD / architecture 원칙과의 cross-link

본 영역이 정의하지 않는 것:
- Spring framework 자체 / 버전 정책 / Kotlin 비교 — [[../spring]]
- DB 자체 (튜닝 / 인덱스 / 백업) — [[../../../database/database]]
- 컨테이너 / 배포 — [[../../../devops/devops]]
- OOP / SOLID 원칙 — [[../../../computer-science/object-oriented-programming/object-oriented-programming]]

---

## 2. 디렉토리 분류

| 디렉토리 | 진입점 | 다루는 주제 |
| --- | --- | --- |
| [[api-design/api-design|api-design]] | api-design.md | 실전 API 구현 레시피 (19) — auth / 게시판 / 채팅 / 결제 / 검색 / 분산 / 운영 |
| [[api-docs/api-docs|api-docs]] | api-docs.md | OpenAPI / Swagger / springdoc |
| [[common/response-envelope|common]] | (디렉토리 직접) | 모든 레시피 공통 — response envelope / security config |
| [[database/database|database]] | database.md | JPA / MyBatis / 두 ORM 공존 / [[database/flyway|Flyway 마이그레이션]] |
| [[observability/observability|observability]] | observability.md | 로그 (Logback / MDC / ELK) + 메트릭 (Actuator / Prometheus / Grafana / Discord Alert) 처음부터 끝까지 |
| [[pitfalls/pitfalls|pitfalls]] | pitfalls.md | n+1 / null safety / [[pitfalls/spring-initializr-version\|Initializr 버전]] / 자주 부딪히는 함정 |

---

## 3. 학습 순서 권장

```
[1. OOP 기초]                       [4. 운영 / 분산]
    ↓                                    ↑
[2. 공통 설정]  →  [3. 첫 API]  →  [3.5 추가 레시피]
                                         ↑
                              [3-1 데이터 접근 (JPA / MyBatis)]
```

| 단계 | 진입점 |
| --- | --- |
| 1. OOP / SOLID / DDD 기초 | [[../../../computer-science/object-oriented-programming/oop-for-backend]] + [[../../../computer-science/object-oriented-programming/ddd-tactical-patterns]] |
| 2. 공통 설정 | [[common/response-envelope]] + [[common/security-config]] |
| 3. 첫 API — signup (가장 단순한 entry-point) | [[api-design/signup/signup]] |
| 3-1. 데이터 접근 — JPA / MyBatis | [[database/database]] → [[database/jpa]] · [[database/mybatis]] |
| 3-2. n+1 / null safety 함정 | [[pitfalls/pitfalls]] |
| 3.5 추가 레시피 | §5 의 19 레시피 중 선택 |
| 4. 운영 — 분산 락 / rate limit / cache / API 문서화 | [[api-design/distributed-lock]] · [[api-design/rate-limiting]] · [[api-design/cache-redis]] · [[api-docs/swagger-springdoc]] |
| 5. 도메인 깊이 | [[../../../computer-science/object-oriented-programming/ddd-tactical-patterns]] + [[../../../40-patterns/architecture-patterns/architecture-patterns]] |

---

## 4. 공통 / 데이터 / 함정 / API 문서

### 4.1 공통 (common)

| 노트 | 다루는 주제 |
| --- | --- |
| [[common/response-envelope]] | API 응답 표준 envelope (성공 / 실패 / 페이지네이션) |
| [[common/security-config]] | Spring Security 설정 (CSRF / CORS / 인증 chain / RBAC) |

→ 모든 api-design 레시피가 이 두 노트를 전제로 작성됨.

### 4.2 데이터 (database)

| 노트 | 다루는 주제 |
| --- | --- |
| [[database/database]] | hub — JPA vs MyBatis 선택 / Hikari / 데이터소스 |
| [[database/jpa]] | JPA 사용 패턴 |
| [[database/mybatis]] | MyBatis 사용 패턴 |
| [[database/jpa-mybatis-coexist]] | 두 ORM 공존 시 가이드 (트랜잭션 / mapper / Entity 분리) |

→ DB 자체 (인덱스 / 튜닝 / 백업) 는 [[../../../database/database]].

### 4.3 함정 (pitfalls)

| 노트 | 다루는 주제 |
| --- | --- |
| [[pitfalls/pitfalls]] | hub — 자주 부딪히는 함정 인덱스 |
| [[pitfalls/n-plus-one]] | JPA n+1 — 진단 + fetch join / @EntityGraph / batch size |
| [[pitfalls/null-safety]] | Optional / `@Nullable` / `@NonNull` / Lombok 의 함정 |

### 4.4 API 문서화 (api-docs)

| 노트 | 다루는 주제 |
| --- | --- |
| [[api-docs/api-docs]] | hub — OpenAPI / Swagger 운영 |
| [[api-docs/swagger-springdoc]] | springdoc-openapi 설정 / customizer / 보안 |

---

## 5. api-design 레시피 인덱스 (19)

각 레시피는 표준 8 섹션 구조 (overview / requirements / domain-model / database / architecture / security / implementation / testing / pitfalls / transactions / prerequisites / operations / implementation-order). 자세한 구조는 [[api-design/api-design]].

### 5.1 Tier 1 — 인증 / 인가 (auth)

| 레시피 | 노트 | 다루는 주제 |
| --- | --- | --- |
| signup | [[api-design/signup/signup]] | 회원가입 — 이메일 인증 / 패스워드 hash / 중복 검증 |
| 2FA | [[api-design/two-factor-auth]] | TOTP / SMS 2단계 인증 |
| RBAC | [[api-design/rbac-permissions]] | 역할 / 권한 / 정책 enforcement |

### 5.2 Tier 2 — 표준 CRUD 도메인

| 레시피 | 노트 | 다루는 주제 |
| --- | --- | --- |
| 게시판 | [[api-design/board/board]] | board 도메인 (글 / 댓글 / 좋아요 / 페이지네이션) |
| 상품 | [[api-design/product/product]] | 상품 도메인 (재고 / 옵션 / 카테고리) |
| 리뷰 | [[api-design/review]] | 별점 / 리뷰 / 신고 |
| 알림 | [[api-design/notification/notification]] | email / push / SMS 멀티 채널 |
| 채팅 | [[api-design/chat/chat]] | 1:1 / 그룹 / 메시지 영속 |

### 5.3 Tier 3 — 검색 / 추천 / 피드

| 레시피 | 노트 | 다루는 주제 |
| --- | --- | --- |
| Elasticsearch 통합 | [[api-design/elasticsearch-integration]] | 인덱싱 / 검색 / 동기화 |
| Geo 검색 | [[api-design/geo-search]] | 위치 기반 검색 (PostGIS / Redis geo) |
| 추천 | [[api-design/recommendation]] | item-based / collaborative / hybrid |
| 피드 / 타임라인 | [[api-design/feed-timeline]] | fan-out write / read / hybrid |

### 5.4 Tier 4 — 통신 / 파일

| 레시피 | 노트 | 다루는 주제 |
| --- | --- | --- |
| WebSocket / STOMP | [[api-design/websocket-stomp]] | 실시간 통신 (채팅 / 알림) |
| Webhook 송신 | [[api-design/webhook-send]] | 이벤트 발신 / 재시도 / 서명 |
| 파일 업로드 (S3) | [[api-design/file-upload-s3]] | presigned URL / multipart / 보안 |
| Excel / CSV import / export | [[api-design/excel-csv-import-export]] | 대용량 / streaming |

### 5.5 Tier 5 — 분산 / 운영

| 레시피 | 노트 | 다루는 주제 |
| --- | --- | --- |
| 분산 락 | [[api-design/distributed-lock]] | Redisson / DB lock / 트랜잭션과의 결합 |
| Rate limiting | [[api-design/rate-limiting]] | token bucket / sliding window / Bucket4j |
| Redis cache | [[api-design/cache-redis]] | cache aside / TTL / eviction |

---

## 6. 결정 매트릭스 — 새 기능 시작

| 결정 | 옵션 |
| --- | --- |
| ORM | JPA (기본, 도메인 중심) / MyBatis (legacy / 복잡 쿼리) / 공존 ([[database/jpa-mybatis-coexist]]) |
| 응답 포맷 | [[common/response-envelope]] 표준 |
| 인증 | [[common/security-config]] + Tier 1 레시피 |
| 트랜잭션 | Service layer 에서 `@Transactional` — 각 레시피의 `transactions.md` |
| 검증 | DTO 의 `@Valid` + 도메인 invariant 의 constructor 검증 |
| 도메인 모델링 | Aggregate / VO 분리 — [[../../../computer-science/object-oriented-programming/ddd-tactical-patterns]] |
| application 구조 | Layered (단순) / Hexagonal (복잡) — [[../../../40-patterns/architecture-patterns/architecture-patterns]] |
| 비동기 | `@Async` + ThreadPoolTaskExecutor / `CompletableFuture` / WebFlux |
| API 문서 | [[api-docs/swagger-springdoc]] |
| 함정 사전 점검 | [[pitfalls/pitfalls]] |

---

## 7. 인접 영역과의 책임 분리

| 본 영역 | 인접 영역 | 분리 기준 |
| --- | --- | --- |
| Spring + Java 코드 | Spring framework 자체 | [[../spring]] (버전 / 모듈 분류) |
| Spring + Java 코드 | Kotlin + Spring 변형 | [[../../kotlin-spring/kotlin-spring]] |
| JPA / MyBatis | DB 자체 운영 | [[../../../database/database]] |
| Service 설계 | OOP 원칙 | [[../../../computer-science/object-oriented-programming/object-oriented-programming]] |
| Service 설계 | 디자인 패턴 | [[../../../computer-science/design-patterns/design-patterns]] |
| application 내부 구조 | architecture 패턴 (Layered/Hexagonal/Clean) | [[../../../40-patterns/architecture-patterns/architecture-patterns]] |
| Security 설정 | 인증 / 암호 이론 | [[../../../security/security]] + [[../../../computer-science/security-theory/security-theory]] |
| HTTP API | HTTP 프로토콜 / TLS | [[../../../computer-science/network/http/http]] + [[../../../computer-science/network/tls-ssl/tls-ssl]] |

---

## 8. 본 영역의 유지보수

| 트리거 | 갱신 항목 |
| --- | --- |
| 새 레시피 추가 | §5 인덱스 (Tier 분류) + 신규 폴더 |
| 새 공통 노트 (common/) | §4.1 |
| 새 DB 패턴 (database/) | §4.2 |
| 새 함정 (pitfalls/) | §4.3 |
| 결정 매트릭스 변경 | §6 |
| 인접 영역 책임 변경 | §7 |
| 학습 순서 변경 | §3 |

frontmatter `updated_at` ISO-8601 기록.

---

## 9. 외부 참고

| 자료 | 영역 |
| --- | --- |
| Spring docs — https://docs.spring.io/spring-boot/ | 공식 |
| Baeldung — https://www.baeldung.com | 실전 레시피 |
| "Spring in Action" 6th (Walls) | 입문~중급 |
| "Pro Spring 6" (Cosmina) | 고급 |
| "Spring Boot in Practice" (Sharma) | 실전 패턴 |
| OpenRewrite — https://docs.openrewrite.org | Boot 2 → 3 마이그레이션 |

상세 학습 자료 인덱스는 [[../spring]] §8 참고.

---

## 10. 관련

- [[../spring|↑ Spring framework hub]]
- [[../../backend|↑ backend hub]]
- [[../../kotlin-spring/kotlin-spring|↗ Kotlin Spring]]
- [[api-design/api-design|→ api-design 레시피 hub]]
- [[api-docs/api-docs|→ API 문서화]]
- [[common/response-envelope|→ 공통 response envelope]]
- [[common/security-config|→ 공통 security config]]
- [[database/database|→ database hub (JPA/MyBatis)]]
- [[pitfalls/pitfalls|→ 함정 hub]]
- [[../../../computer-science/object-oriented-programming/oop-for-backend|↗ OOP for backend]]
- [[../../../computer-science/object-oriented-programming/ddd-tactical-patterns|↗ DDD tactical patterns]]
- [[../../../40-patterns/architecture-patterns/architecture-patterns|↗ architecture patterns]]
- [[../../../database/database|↗ DB 자체]]
- [[../../../security/security|↗ security]]
