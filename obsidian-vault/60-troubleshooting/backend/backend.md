---
title: "60-troubleshooting/backend — Backend 트러블 카탈로그"
kind: knowledge
status: current
created_at: 2026-05-27
tags: [troubleshooting, backend, jpa, spring, java, python, hub]
home_hub: "60-troubleshooting"
---

# Backend 트러블 카탈로그

> 서버 사이드 (Spring / JPA / Hibernate / Python / Node) 에서 자주 만나는 대표 트러블.
> 깊이 노트는 각 항목에서 별도 파일로 분리.

**[[../troubleshooting|↑ 60-troubleshooting]]** · **[[../../20-areas/backend/backend|↑↑ 20-areas/backend]]**

---

## 카탈로그

| # | 증상 / 에러 | 한 줄 원인 | 핵심 해결 |
| - | --- | --- | --- |
| 1 | **N+1 쿼리** — list API 가 갑자기 100+ SELECT | 연관 엔티티 lazy 접근 in loop | `fetch join` / `@EntityGraph` / `JOIN FETCH` / batch size |
| 2 | **LazyInitializationException** — `could not initialize proxy ... no Session` | 트랜잭션 밖에서 lazy proxy 접근 | DTO 매핑을 트랜잭션 안에서 / `@Transactional(readOnly=true)` controller layer 금지 |
| 3 | **OutOfMemoryError: Java heap space** | 큰 결과 set 을 in-memory 로 로딩 | streaming (`Stream<T>`) / pagination / `ScrollableResults` |
| 4 | **OutOfMemoryError: Metaspace** | classloader leak (devtools / hot-reload) | `-XX:MaxMetaspaceSize` + 명시 GC root 추적 / dev profile 분리 |
| 5 | **HikariPool — Connection is not available** | connection 누수 또는 풀 크기 < concurrent | try-with-resources / `spring.datasource.hikari.leak-detection-threshold=60000` |
| 6 | **TransactionTemplate REQUIRES_NEW 가 무시됨** | self-invocation 으로 AOP proxy 우회 | 다른 bean 으로 추출 / `AopContext.currentProxy()` |
| 7 | **Spring AOP `@Async` / `@Transactional` 동작 안 함** | private method / 같은 bean self-call | public + 다른 bean 추출 / `@EnableAspectJAutoProxy(exposeProxy=true)` |
| 8 | **NullPointerException 의 origin 추적 불가** | Optional 미사용 + 깊은 chaining | `Optional.ofNullable` / `?.` (Kotlin) / Java 14+ helpful NPE message |
| 9 | **JSON serialization fail — `failed to lazily initialize`** | Jackson 이 JPA proxy 직렬화 시도 | DTO 분리 (entity 직접 반환 금지) / `@JsonIgnore` lazy field / `Hibernate5Module` |
| 10 | **Thread pool 고갈 — request 가 멈춤** | blocking IO 가 worker pool 점유 | bounded async pool 분리 / virtual thread (Java 21) / reactive 전환 |
| 11 | **Time zone bug — UTC ↔ KST 9시간 차이** | DB / app / client 의 TZ 불일치 | `serverTimezone=UTC` JDBC URL + `Instant` 사용 / `LocalDateTime` 금지 |
| 12 | **Circular dependency (Bean creation exception)** | A → B → A 직접 의존 | `@Lazy` / 인터페이스 분리 / 책임 재설계 (가능하면 lazy 회피 X) |
| 13 | **DeadlockException — `Deadlock found when trying to get lock`** | 다른 순서로 같은 row 잠금 | 트랜잭션 lock 순서 통일 / 짧게 / `SELECT ... FOR UPDATE` 순서 명시 |
| 14 | **File descriptor leak — `Too many open files`** | InputStream / Socket close 누락 | try-with-resources / `lsof -p <pid>` / `ulimit -n` 모니터 |
| 15 | **Spring Security 401 — `Full authentication is required`** | SecurityFilterChain 가 anon 거부 | `permitAll()` matcher / `WebSecurityConfigurerAdapter` (deprecated) → 새 `SecurityFilterChain` bean |

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
