---
title: "troubleshooting · HikariPool — Connection is not available"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [backend, spring, hikari, jdbc, connection-pool, leak]
home_hub: "60-troubleshooting"
related:
  - "[[backend]]"
  - "[[../database/database|database 트러블]]"
---

# HikariPool — Connection is not available

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 — leak detection / pool sizing / try-with-resources |

## 증상

```
java.sql.SQLTransientConnectionException:
  HikariPool-1 - Connection is not available, request timed out after 30000ms.
    at com.zaxxer.hikari.pool.HikariPool.createTimeoutException(HikariPool.java:696)
```

부가 신호:
```
HikariPool-1 - Pool stats (total=20, active=20, idle=0, waiting=15)
HikariPool-1 - Connection leak detection triggered for ... on thread http-nio-8080-exec-3,
  stack trace follows
    java.lang.Throwable: Apparent connection leak detected
```

- 평소 정상, 부하 spike 시 / 특정 endpoint 호출 후 발생
- 점진적으로 `active` 가 max 까지 증가 → 회복 안 됨

## 가설

- 처음에는 pool size 부족 → max 늘려도 시간만 지연 (leak 이라 결국 다시 max 도달)
- DB 가 느린가 → slow query log 봤지만 평소 수준
- 실제: connection 을 close 안 한 경로가 있음 → leak

## 원인 (root cause)

3가지가 흔함:

**(1) try-with-resources 누락**
```java
// Bad
Connection con = dataSource.getConnection();
PreparedStatement ps = con.prepareStatement("SELECT ...");
ResultSet rs = ps.executeQuery();
return mapToList(rs);  // ← 예외 시 close 안 됨
```

**(2) JPA repository 가 stream 반환 + caller 가 stream close 안 함**
```java
@Query("SELECT o FROM Order o")
Stream<Order> streamAll();

// Caller
Stream<Order> stream = repo.streamAll();  // ← try-with-resources X
stream.forEach(...);
```

**(3) `@Transactional` 안에 blocking IO**

```java
@Transactional
public void process(Long id) {
    var order = repo.findById(id);
    restTemplate.exchange(...);  // ← network call 가 5초 → 그 동안 connection 점유
}
```

→ connection pool 이 사실상 외부 API latency 에 묶임.

## 해결

### 즉시 (workaround)

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 50               # 임시 확대
      connection-timeout: 5000            # 5초 → 빠르게 실패 (장시간 wait 차단)
      leak-detection-threshold: 60000     # 60초 보유 시 stack trace 로그
```

### 근본

**(1) try-with-resources 강제**

```java
// Good
try (var con = dataSource.getConnection();
     var ps = con.prepareStatement(SQL);
     var rs = ps.executeQuery()) {
    return mapToList(rs);
}
```

JPA / Spring Data 사용 시 대부분 framework 가 close 보장 — **직접 `dataSource.getConnection()` 호출 코드만 주의**.

**(2) Stream 사용 시 caller 가 try-with-resources**

```java
try (Stream<Order> stream = repo.streamAll()) {
    stream.forEach(this::process);
}
```

**(3) Transaction 안에 IO 금지**

```java
// Bad
@Transactional
public void process(Long id) {
    var order = repo.findById(id);
    restTemplate.exchange(...);  // ← 외부 호출
    repo.save(order);
}

// Good — transaction 분리
public void process(Long id) {
    var order = readInTx(id);
    var external = restTemplate.exchange(...);  // tx 밖
    writeInTx(order, external);
}

@Transactional(readOnly = true)
public Order readInTx(Long id) { ... }

@Transactional
public void writeInTx(Order o, ExternalDto e) { ... }
```

원칙: **트랜잭션 안에서 외부 시스템 호출 금지** — 트랜잭션 시간 = connection 점유 시간.

### Pool sizing 가이드 (HikariCP wiki)

```
pool size = ((core_count × 2) + effective_spindle_count)
```

- 일반 OLTP: **CPU × 2 + 1** 정도
- 더 키운다고 빨라지지 X — DB context switch / lock 경쟁만 증가
- 큰 web 앱이 pool 100 가는 건 거의 안티패턴 (sharding / async / read replica 검토)

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20         # CPU 8 core → 17 정도가 상한
      minimum-idle: 10
      connection-timeout: 3000      # 3초
      idle-timeout: 600000          # 10분
      max-lifetime: 1800000         # 30분 (DB wait_timeout 보다 짧게)
      leak-detection-threshold: 60000
      validation-timeout: 5000
```

### Leak detection 활용

`leak-detection-threshold` 가 작동하면 누가 보유 중인지 stack trace 로그:

```
Apparent connection leak detected
  at com.example.LegacyDao.find(LegacyDao.java:42)
  at com.example.OrderService.process(OrderService.java:88)
```

→ 그 경로 try-with-resources 보강.

## 진단 명령

```bash
# Hikari metrics (Micrometer)
hikaricp_connections_active
hikaricp_connections_idle
hikaricp_connections_pending
hikaricp_connections_timeout_total

# DB side (MySQL)
SHOW PROCESSLIST;
SELECT * FROM information_schema.innodb_trx;

# DB side (Postgres)
SELECT pid, state, query, query_start FROM pg_stat_activity
  WHERE state != 'idle' ORDER BY query_start;
```

## 재발 방지

- **`leak-detection-threshold = 60000`** 을 prod 에도 켜기 (overhead 무시 가능)
- **Spotbugs / ErrorProne** 으로 `Connection`/`Statement` close 누락 검출
- **Architecture rule (ArchUnit)**: `@Transactional` 메서드 안에서 `RestTemplate` / `WebClient` 호출 금지
  ```java
  @Test
  void transactionalMethodsShouldNotCallRestTemplate() {
      noMethods().that().areAnnotatedWith(Transactional.class)
        .should().callMethodWhere(target("RestTemplate"));
  }
  ```
- **부하 테스트**: k6 sustained load → `hikaricp_connections_active` 그래프 평탄성 검증
- **알람**:
  ```
  - alert: HikariSaturated
    expr: hikaricp_connections_active / hikaricp_connections_max > 0.9
    for: 2m
  ```

## 관련

- [[backend]]
- [[../database/database|database 트러블]] — connection 고갈은 양쪽 공통
- [[thread-pool-exhausted]] — connection pool 과 thread pool 의 cascading 고갈
- 책: 자바 성능 튜닝 / HikariCP wiki
