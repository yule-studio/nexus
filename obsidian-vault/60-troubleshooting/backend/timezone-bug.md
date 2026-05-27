---
title: "troubleshooting · Time zone bug — UTC ↔ KST 9시간 차이"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [backend, java, datetime, jdbc, mysql, timezone]
home_hub: "60-troubleshooting"
related:
  - "[[backend]]"
---

# Time zone bug — UTC ↔ KST 9시간 어긋남

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 — Instant / serverTimezone / Jackson |

## 증상

- DB 의 `created_at = 2026-05-27 09:00:00` 인데 API 응답에서 `2026-05-27T00:00:00Z`
- 또는 반대로 KST 입력이 UTC 로 저장 → 다음날 표시
- 자정 정산 / 일별 통계가 ±1 일 차이
- Daylight saving (KST 는 없지만 미국 / 유럽 사용자) 시즌 전후 1시간 차이

## 가설

- 코드 어딘가 `+9 hour` 직접 더하면? → fragile
- DB column 을 `DATETIME` 가 아닌 `TIMESTAMP` 로? → JDBC driver 가 또 다르게 해석
- 실제: app / JDBC / DB 의 TZ 가 명시되지 않아 각자 가정 다름

## 원인 (root cause)

5 계층 모두 TZ 가 있고 default 가 다름:

```
1. JVM             — TimeZone.getDefault() (보통 OS 기본 = container UTC 가 흔함)
2. JDBC driver     — serverTimezone parameter
3. DB server       — global time_zone (MySQL) / TimeZone (Postgres)
4. DB column       — TIMESTAMP (TZ aware) vs DATETIME (no TZ)
5. Jackson         — ObjectMapper.setTimeZone()
```

특히 MySQL 의 함정:

```
TIMESTAMP — JDBC driver 가 session TZ 로 UTC ↔ session 변환
DATETIME  — TZ 무관 (저장된 값 그대로)
```

→ `DATETIME` column 에 KST 시각 저장한 후 driver TZ 가 UTC 면 → app 은 UTC 로 해석 → 9시간 어긋남.

## 해결

### 즉시 (workaround) — 절대 사용 X

```java
// Bad — 임시 shift
return Instant.now().plus(9, HOURS);
```

→ DST 가 있는 지역에서 깨짐. 영구 결함.

### 근본 (원칙 + 5계층 통일)

**원칙**: **저장 / 전송은 UTC (`Instant`), 표시만 사용자 TZ.**

#### 1. JVM 통일 — UTC

```dockerfile
ENV TZ=UTC
RUN ln -sf /usr/share/zoneinfo/$TZ /etc/localtime
```

또는 JVM:
```
-Duser.timezone=UTC
```

#### 2. JDBC URL

```
# MySQL
jdbc:mysql://host:3306/db?serverTimezone=UTC&useLegacyDatetimeCode=false

# Postgres — 보통 자동, 명시:
jdbc:postgresql://host:5432/db?stringtype=unspecified
```

#### 3. DB server

```sql
-- MySQL
SET GLOBAL time_zone = '+00:00';

-- Postgres
ALTER DATABASE mydb SET timezone TO 'UTC';
```

#### 4. Column type — `TIMESTAMP WITH TIME ZONE` (Postgres) / `TIMESTAMP` (MySQL)

```sql
-- MySQL
CREATE TABLE order (
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP   -- TZ aware
);

-- Postgres
CREATE TABLE "order" (
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**MySQL `DATETIME` 사용 금지** (또는 의도 분명할 때만) — TZ 정보 없음 → 명시 안 한 가정으로 깨짐.

#### 5. Java type — `Instant` 우선

```java
@Entity
public class Order {
    @Column(name = "created_at")
    private Instant createdAt;        // ← 항상 UTC 의 순간

    // LocalDateTime 은 TZ 없음 → 위험
    // ZonedDateTime / OffsetDateTime 는 명시적이지만 Instant 보다 무거움
}
```

JPA / Hibernate 가 `Instant ↔ TIMESTAMP` 자동 변환 (UTC 기준).

#### 6. Jackson serialization

```java
@Bean
public Jackson2ObjectMapperBuilderCustomizer customizer() {
    return b -> b
        .timeZone(TimeZone.getTimeZone("UTC"))
        .featuresToDisable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
        .modules(new JavaTimeModule());
}
```

→ `Instant` 가 ISO-8601 `2026-05-27T00:00:00Z` 로 직렬화.

#### 7. 사용자 표시는 마지막에 변환

```java
ZonedDateTime userView = order.getCreatedAt().atZone(ZoneId.of("Asia/Seoul"));
String formatted = userView.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm"));
```

### 자정 정산 / 일별 통계 함정

```java
// Bad — 서버 TZ 기준
LocalDate today = LocalDate.now();   // server TZ 의 today

// Good — 사용자 TZ 기준
LocalDate todayKr = LocalDate.now(ZoneId.of("Asia/Seoul"));

// DB 조회 (UTC 저장된 createdAt 을 KST 자정 기준 필터)
ZoneId kr = ZoneId.of("Asia/Seoul");
Instant from = todayKr.atStartOfDay(kr).toInstant();
Instant to = from.plus(1, DAYS);
orderRepo.findByCreatedAtBetween(from, to);
```

## 진단

```sql
-- MySQL
SELECT @@global.time_zone, @@session.time_zone, NOW(), UTC_TIMESTAMP();

-- Postgres
SHOW timezone;
SELECT now(), now() AT TIME ZONE 'UTC';
```

```java
// JVM
System.out.println(TimeZone.getDefault());   // UTC 권장
System.out.println(Instant.now());           // ISO 형식
System.out.println(LocalDateTime.now());     // server local — 위험 신호

// JDBC 연결 후
try (var con = ds.getConnection()) {
    var rs = con.createStatement().executeQuery("SELECT @@session.time_zone");
    rs.next();
    System.out.println(rs.getString(1));
}
```

## 재발 방지

- **container 표준**: 모든 image base 에 `ENV TZ=UTC`
- **lint 룰**: `LocalDateTime.now()` / `Date` 사용 시 warning
  ```java
  // archunit
  noClasses().should().callMethodWhere(target("LocalDateTime.now()"));
  ```
- **단위 테스트**: TZ 변경 시뮬레이션
  ```java
  @ParameterizedTest
  @ValueSource(strings = {"UTC", "Asia/Seoul", "America/New_York"})
  void shouldHandleTimezones(String tz) {
      TimeZone.setDefault(TimeZone.getTimeZone(tz));
      // ... test
  }
  ```
- **PR 룰**: `Date` / `Calendar` / `LocalDateTime` 신규 등장 reject (`Instant` / `ZonedDateTime` 권장)

## 관련

- [[backend]]
- [[../database/database|database 트러블]] — DB TZ 영역
- 책: Effective Java - Item 84 / Java 8 Date Time API
