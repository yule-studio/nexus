---
title: "60-troubleshooting/backend — Backend 트러블 카탈로그"
kind: knowledge
status: current
created_at: 2026-05-27
tags: [troubleshooting, backend, jpa, spring, java, python, hub]
home_hub: "60-troubleshooting"
---

# Backend 트러블 카탈로그

> 서버 사이드 (Java / Spring / JPA / Hibernate) 에서 자주 만나는 대표 트러블.
> 각 항목은 별도 깊이 노트로 분리 — Java + Spring 컨텍스트.

**[[../troubleshooting|↑ 60-troubleshooting]]** · **[[../../20-areas/backend/backend|↑↑ 20-areas/backend]]**

---

## 카탈로그

| #  | 증상 / 에러                                                                    | 한 줄 원인                              | 핵심 해결                                                       | 깊이 노트                                            |
| -- | -------------------------------------------------------------------------- | ---------------------------------- | ----------------------------------------------------------- | ------------------------------------------------ |
| 1  | **N+1 쿼리** — list API 가 갑자기 100+ SELECT                                    | 연관 엔티티 lazy 접근 in loop             | `fetch join` / `@EntityGraph` / `@BatchSize` / default_batch_fetch_size | [[n-plus-one-query]]                             |
| 2  | **LazyInitializationException** — `could not initialize proxy ... no Session` | 트랜잭션 밖에서 lazy proxy 접근             | DTO 매핑을 트랜잭션 안에서 / OSIV=false / fetch join                  | [[lazy-initialization-exception]]                |
| 3  | **OutOfMemoryError: Java heap space**                                      | 큰 결과 set 을 in-memory 로 로딩          | Stream + pagination / `em.clear()` 주기 / StreamingResponseBody | [[oom-heap]]                                     |
| 4  | **OutOfMemoryError: Metaspace**                                            | classloader leak (devtools / hot-reload) | `-XX:MaxMetaspaceSize` + MAT 분석 / ThreadLocal cleanup / dev profile 분리 | [[oom-metaspace]]                                |
| 5  | **HikariPool — Connection is not available**                               | connection 누수 또는 트랜잭션 안 IO         | try-with-resources / leak-detection-threshold / 트랜잭션 안 외부 호출 금지 | [[hikari-connection-leak]]                       |
| 6  | **@Transactional(REQUIRES_NEW) 가 무시됨**                                     | self-invocation 으로 AOP proxy 우회    | 다른 bean 으로 분리 / TransactionTemplate / @TransactionalEventListener | [[transaction-requires-new-ignored]]             |
| 7  | **Spring AOP `@Async` / `@Cacheable` 동작 안 함**                              | private method / 같은 bean self-call  | public + 다른 bean 추출 / AspectJ weaving / `AopContext.currentProxy()` | [[aop-self-invocation]]                          |
| 8  | **NullPointerException 의 origin 추적 불가**                                    | Optional 미사용 + 깊은 chaining          | helpful NPE (`-XX:+ShowCodeDetailsInExceptionMessages`) / Optional / @NonNullApi | [[npe-deep-chain]]                               |
| 9  | **Jackson — `failed to lazily initialize`**                                | Jackson 이 JPA proxy 직렬화 시도         | DTO 분리 (entity 노출 금지) / MapStruct / JPQL constructor projection | [[jackson-lazy-proxy]]                           |
| 10 | **Thread pool 고갈 — request 가 멈춤**                                          | blocking IO 가 worker pool 점유       | virtual thread (Java 21) / @Async 분리 pool / RestTemplate timeout | [[thread-pool-exhausted]]                        |
| 11 | **Time zone bug — UTC ↔ KST 9시간 차이**                                       | DB / app / client 의 TZ 불일치          | container `TZ=UTC` + `serverTimezone=UTC` + `Instant` 사용    | [[timezone-bug]]                                 |
| 12 | **Circular dependency (Bean creation exception)**                          | A → B → A 직접 의존                    | 책임 분리 / ApplicationEvent / @Lazy (최후) / ObjectProvider     | [[circular-dependency]]                          |
| 13 | **DeadlockException — `Deadlock found when trying to get lock`**           | 다른 순서로 같은 row 잠금                   | lock 순서 통일 / @Retryable / 짧은 트랜잭션 / pessimistic lock      | [[deadlock-mysql]]                               |
| 14 | **File descriptor leak — `Too many open files`**                           | InputStream / Socket close 누락       | try-with-resources / HttpClient connection pool / Stream close | [[file-descriptor-leak]]                         |
| 15 | **Spring Security 401 — `Full authentication is required`**                | SecurityFilterChain default deny    | 새 SecurityFilterChain bean / OPTIONS permitAll / JWT issuer 검증 | [[spring-security-401]]                          |

## 자주 쓰는 진단 명령

```bash
# JVM heap dump
jmap -dump:format=b,file=heap.bin <pid>

# Thread dump
jstack <pid> > thread.txt

# GC log
-Xlog:gc*:file=gc.log:time,uptime,level,tags

# 활성 connection
SELECT * FROM information_schema.processlist WHERE COMMAND != 'Sleep';
```

## 관련

- [[../troubleshooting]]
- [[../../20-areas/backend/backend|20-areas/backend]]
- [[../database/database|database 트러블]] — JDBC / Hikari 는 backend ↔ database 경계
- yule-studio-agent: `hookify` mistake_ledger
