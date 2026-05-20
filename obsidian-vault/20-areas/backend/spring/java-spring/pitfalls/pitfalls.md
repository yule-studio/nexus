---
title: "Java Spring — 유명 이슈 / 함정 Hub"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T13:00:00+09:00
tags:
  - backend
  - java-spring
  - pitfalls
  - hub
---

# Java Spring — 유명 이슈 / 함정 Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | hub — 8 카테고리 함정 카탈로그 |
| v.1.1.0 | 2026-05-20 | engineering-agent/tech-lead | Spring Security 401 vs 403 함정 추가 (bkurs-be B2A 회수) |
| v.1.2.0 | 2026-05-20 | engineering-agent/tech-lead | @Transactional default rollback noRollbackFor 함정 추가 (bkurs-be B2B 회수) + 인증/권한 카테고리 |

**[[../java-spring|↑ Java Spring]]**

> 이 폴더는 **Java + Spring 백엔드에서 한 번은 다 겪는 함정들**.
> 각 노트는 **증상 → 원인 → 감지 방법 → 해결 → 코드 예제 → 측정**.
> 레시피 (`api-design/`) 가 "어떻게 만드나" 라면, pitfalls 는 "왜 안 되나 / 망가지나".

---

## 0. 8개 카테고리

| # | 카테고리 | 노트 | 한 줄 |
| --- | --- | --- | --- |
| 1 | **쿼리 / ORM** | [[n-plus-one]] | 가장 흔한 성능 사고. 목록 = N+1 쿼리. |
| 2 | **null / 타입 안정성** | [[null-safety]] | NPE 천국. Optional / @Nullable / record. |
| 3 | **트랜잭션 / 락** | [[transaction-pitfalls]] · [[transactional-rollback-noRollbackFor]] | self-invocation / lazy outside session / 긴 트랜잭션 / default rollback 이 부수효과 묻음. |
| 4 | **동시성** | [[concurrency-pitfalls]] | race / lost update / 낙관·비관·분산. |
| 5 | **캐시** | [[cache-stampede]] | hot key, dog-pile, thundering herd. |
| 6 | **메모리 / 리소스** | [[resource-leaks]] | connection pool / file handle / OOM / GC. |
| 7 | **시간 / 인코딩** | [[time-encoding]] | timezone / DST / utf8mb4 / 이모지 / NFC. |
| 8 | **에러 / 로깅** | error-handling-pitfalls (예정) | 삼킨 예외 / 평문 로깅 / @Async exception. |
| 9 | **인증 / 권한** | [[spring-security-401-vs-403]] | Spring Security default 가 인증 실패도 403 — REST API 면 401 명시 필수. |

---

## 1. 가장 자주 사고 — 발생 빈도 순 (개인 통계 / 일반론)

1. **N+1** — ORM 쓰는 거의 모든 프로젝트에서 한 번은 사고
2. **lazy loading outside session** — `LazyInitializationException`
3. **NPE** — 외부 API 응답 / DB 컬럼 null
4. **@Transactional self-invocation** — 같은 클래스의 메서드 호출 → AOP 미작동
5. **시간대 (Asia/Seoul vs UTC) 혼동** — DB 저장은 UTC, 표시는 KST 인데 어디서 변환?
6. **utf8 (3-byte) vs utf8mb4 (4-byte)** — MySQL 5.5 이하 default 가 이모지 저장 실패
7. **`@Async` / `@Scheduled` 의 예외 silent**
8. **connection pool exhaustion** — 외부 API 동기 호출 + timeout 안 걸음
9. **race condition / lost update** — `findById → 수정 → save` 사이 다른 트랜잭션
10. **N+1 의 사촌 — cartesian product** — `@OneToMany` fetch join 둘

---

## 2. 진단 도구 (한국 / 글로벌 공통)

| 영역 | 도구 |
| --- | --- |
| SQL 추적 | `p6spy` / `datasource-proxy` / Hibernate `show_sql` + formatter |
| Hibernate stat | `hibernate.generate_statistics=true` + `Statistics` API |
| APM | DataDog / New Relic / Pinpoint (한국) / SkyWalking |
| Java profiling | async-profiler / JFR (Java Flight Recorder) |
| GC | `-Xlog:gc*` / GCViewer / GCeasy.io |
| Heap dump | `jmap` / `jcmd` / MAT (Eclipse Memory Analyzer) |
| Thread dump | `jstack` / Fastthread.io |
| DB plan | PG `EXPLAIN ANALYZE` / MySQL `EXPLAIN FORMAT=JSON` |
| Connection pool | HikariCP `metricsTrackerFactory` + Micrometer |
| Distributed tracing | OpenTelemetry / Zipkin / Jaeger |

---

## 3. 운영 직전 체크리스트 (모든 프로젝트)

- [ ] Hibernate `generate_statistics=true` (dev / staging)
- [ ] SQL 로그 (DEBUG 이하) — 운영은 INFO
- [ ] HikariCP `maximumPoolSize`, `connectionTimeout` 명시
- [ ] 모든 외부 HTTP 호출에 timeout (connect / read)
- [ ] 모든 `@Transactional` 의 `readOnly` 명시
- [ ] `@Async` 메서드의 예외 처리 (`AsyncUncaughtExceptionHandler`)
- [ ] DB 컬럼 `NOT NULL` 명시 (JPA `nullable = false`)
- [ ] `application.yml` 의 timezone 명시 (`spring.jackson.time-zone`, `JVM_TZ=UTC`)
- [ ] MySQL charset `utf8mb4` (UTF-8 4-byte 이모지)
- [ ] Heap / Metaspace / GC 모니터링
- [ ] 분산 트레이싱 (서비스 ≥ 2 개)

---

## 4. 노트 진행 상태

| 노트 | 상태 |
| --- | --- |
| [[n-plus-one]] | ✅ |
| [[null-safety]] | ✅ |
| [[transaction-pitfalls]] | 🟡 |
| [[concurrency-pitfalls]] | 🟡 |
| [[cache-stampede]] | 🟡 |
| [[resource-leaks]] | 🟡 |
| [[time-encoding]] | 🟡 |
| error-handling-pitfalls | 🟡 |
| [[spring-security-401-vs-403]] | ✅ |
| [[transactional-rollback-noRollbackFor]] | ✅ |

---

## 5. 관련

- [[../api-design/api-design|↗ Java Spring API 레시피]]
- [[../../../../database/database|↗ database hub]]
- [[../java-spring|↑ Java Spring]]
