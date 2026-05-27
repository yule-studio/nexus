---
title: "troubleshooting · NullPointerException origin 추적"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [backend, java, npe, optional, jpa]
home_hub: "60-troubleshooting"
related:
  - "[[backend]]"
  - "[[jackson-lazy-proxy]]"
---

# NullPointerException — origin 추적이 어려운 deep chain

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 — helpful NPE / Optional / @NonNull |

## 증상

```
java.lang.NullPointerException
    at com.example.OrderService.process(OrderService.java:42)
```

Java 13 이하 stack trace 는 "어느 변수" 가 null 인지 알려주지 않음.

```java
// 어디서 null?
return order.getCustomer().getAddress().getZipCode();
```

## 가설

- DB row 가 비었나 → 데이터 보강해도 다른 곳에서 또
- 로그 추가 → 매 commit 마다 if 분기 늘어남
- 근본: null 의 출처를 컴파일 / 정적 분석 단계에서 잡지 못함

## 원인 (root cause)

Java 의 null 은 type 정보가 없음. nullable 의도가 코드에 표현돼 있지 않으면 caller / callee 가 다른 가정으로 작성됨.

```java
// Bad — null 의도 불명확
public Address findAddress(Long id) { ... return null; }   // 가끔 null

// Caller 가 모름
findAddress(id).getZipCode();  // → NPE
```

## 해결

### 즉시 — helpful NPE 활성화 (Java 14+)

```
-XX:+ShowCodeDetailsInExceptionMessages    # Java 14+ default 인데 명시
```

```
java.lang.NullPointerException:
  Cannot invoke "Address.getZipCode()" because the return value of "Customer.getAddress()" is null
    at com.example.OrderService.process(OrderService.java:42)
```

→ 어느 객체 / 어느 호출에서 null 인지 즉시 보임.

### 근본 (4가지)

**(1) Optional 명시**

```java
public Optional<Address> findAddress(Long id) {
    return Optional.ofNullable(jdbcTemplate.queryForObject(...));
}

// Caller
String zip = orderRepo.findById(id)
    .flatMap(o -> Optional.ofNullable(o.getCustomer()))
    .flatMap(c -> findAddress(c.getId()))
    .map(Address::getZipCode)
    .orElse("UNKNOWN");
```

원칙: **반환에 Optional, 인자에는 Optional 사용 X** — 인자는 `@NonNull` / overload 로 표현.

**(2) `@NonNull` / `@Nullable` 정적 분석**

```java
import org.springframework.lang.NonNull;
import org.springframework.lang.Nullable;

public class CustomerService {
    @NonNull
    public Customer find(@NonNull Long id) { ... }   // null 절대 반환 X

    @Nullable
    public Customer findOrNull(@NonNull Long id) { ... }
}
```

IntelliJ / SonarQube / SpotBugs 가 caller 에서 null check 누락 경고. `package-info.java` 에 default 지정:

```java
@NonNullApi
@NonNullFields
package com.example;
```

→ 패키지 전체 default `@NonNull`. nullable 만 `@Nullable` 명시.

**(3) Kotlin** (가능하면)

```kotlin
fun zipFor(order: Order): String =
    order.customer?.address?.zipCode ?: "UNKNOWN"
```

null 이 type system level. compile-time 차단.

**(4) Builder + 필수 인자 검증**

```java
@Value
@Builder
public class Order {
    @NonNull Long id;
    @NonNull Customer customer;

    // Lombok @NonNull → 생성 시 null 이면 NPE 즉시 (lazy NPE 회피)
}
```

## JPA 관련 NPE 특수 케이스

**(1) `findById` 가 Optional.empty 인데 `.get()` 직접 호출**

```java
// Bad
Order o = repo.findById(id).get();   // NoSuchElementException (NPE 아니지만 비슷)

// Good
Order o = repo.findById(id).orElseThrow(() -> new OrderNotFound(id));
```

**(2) JPA entity 의 `@ManyToOne` lazy proxy 가 null 인 row → toString / equals 에서 NPE**

```java
@Entity
public class Order {
    @ManyToOne(fetch = LAZY)
    private Customer customer;     // optional=true 가 default

    // Lombok @ToString 가 customer.toString() → proxy 가 null 이면 NPE
}
```

→ `@ToString(exclude = "customer")` 또는 `@ManyToOne(optional = false)` 명시.

**(3) `@Column(nullable = false)` 가 java field 와 불일치**

```java
@Column(nullable = false)
private String name;   // DB 는 NOT NULL 이지만 Java 는 null 허용
```

→ Lombok `@NonNull` 추가 또는 record 사용.

## 진단

```bash
# Helpful NPE 켜기 (default Java 15+)
-XX:+ShowCodeDetailsInExceptionMessages

# 정적 분석
mvn spotbugs:check
mvn pmd:check
mvn dependency:analyze

# IntelliJ inspection
# Settings → Editor → Inspections → Probable bugs → "@NotNull/@Nullable problems"
```

## 재발 방지

- **`-XX:+ShowCodeDetailsInExceptionMessages`** 운영도 ON (overhead 거의 없음)
- **`@NonNullApi @NonNullFields`** package-info 적용 (점진적)
- **CI**: spotbugs / pmd / SonarQube 가 null safety warning 시 FAIL
- **PR 룰**: `null` 반환 추가 시 명시적 javadoc + caller 변경 필수
- **회귀 테스트**: NPE 가 났던 경로의 null edge case 테스트 추가

## 관련

- [[backend]]
- [[jackson-lazy-proxy]] — JSON serialization 시 NPE
- [[lazy-initialization-exception]] — JPA 영역
- 책: Effective Java 3rd Ed - Item 55 (Optional)
