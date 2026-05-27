---
title: "troubleshooting · Spring AOP self-invocation 무력화"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [backend, spring, aop, async, cacheable, proxy]
home_hub: "60-troubleshooting"
related:
  - "[[backend]]"
  - "[[transaction-requires-new-ignored]]"
---

# Spring AOP self-invocation — @Async / @Cacheable / @Transactional 가 작동 안 함

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 — proxy 메커니즘 / AspectJ / bean 분리 |

## 증상

```java
@Service
public class NotificationService {

    public void send(User u) {
        sendEmail(u);          // ← @Async 무시 → 동기 실행
    }

    @Async
    public void sendEmail(User u) { ... }
}
```

- `@Async` 무시 → 같은 thread 에서 실행
- `@Cacheable` 무시 → 매번 실제 메서드 호출
- `@PreAuthorize` 무시 → 권한 체크 안 됨
- `@Retryable` (spring-retry) 무시 → 재시도 안 됨

→ 보안 / 성능 / 안정성 모두 영향.

## 가설

- annotation 적용 빠뜨림? → 명백히 있음
- bean 등록 누락? → context 에 정상 등록
- 실제: self-invocation (같은 클래스 안에서 `this.method()` 호출)

## 원인 (root cause)

Spring 의 대부분 AOP-based annotation 은 **JDK Dynamic Proxy** 또는 **CGLIB proxy** 로 구현. proxy 가 메서드 호출을 가로채 advice 적용.

```
외부 caller → proxy.foo() → [advice] → target.foo()
                                          ↓ this.bar()  ← proxy 거치지 않음
                                            (직접 호출)
```

`this` 는 target 객체이지 proxy 객체가 아님. 그래서 `this.bar()` 는 proxy advice 우회 → annotation 무시.

영향받는 annotation:
- `@Transactional`
- `@Async`
- `@Cacheable` / `@CacheEvict` / `@CachePut`
- `@PreAuthorize` / `@PostAuthorize`
- `@Retryable` (spring-retry)
- `@CircuitBreaker` (resilience4j-spring)
- `@Validated` method-level
- 모든 custom `@Aspect` advice

## 해결

### 즉시 (workaround) — proxy 강제 호출

```java
@Configuration
@EnableAspectJAutoProxy(exposeProxy = true)  // 핵심
class Config {}

@Service
public class NotificationService {
    public void send(User u) {
        ((NotificationService) AopContext.currentProxy()).sendEmail(u);  // proxy 경유
    }

    @Async
    public void sendEmail(User u) { ... }
}
```

작동하나 결합도 ↑. 임시만.

### 근본 (3가지)

**(1) 별도 bean 으로 분리** (권장)

```java
@Service
public class NotificationService {
    private final EmailSender emailSender;

    public void send(User u) {
        emailSender.send(u);  // ← 다른 bean → proxy 경유
    }
}

@Component
public class EmailSender {
    @Async
    public void send(User u) { ... }
}
```

**(2) AspectJ Compile-time / Load-time weaving** — proxy 한계 자체 회피

```xml
<dependency>
  <groupId>org.aspectj</groupId>
  <artifactId>aspectjweaver</artifactId>
</dependency>
```

```java
@EnableLoadTimeWeaving(aspectjWeaving = ENABLED)
@EnableAspectJAutoProxy(proxyTargetClass = true)
```

→ proxy 없이 bytecode 변경 → self-invocation 도 advice 적용. 다만 build/runtime 복잡도 ↑.

**(3) Spring 6 가상 thread (`@Async` 대안)** — Java 21

```java
@Bean
public AsyncTaskExecutor applicationTaskExecutor() {
    return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
}
```

self-invocation 문제는 그대로지만 sender bean 분리 시 가상 thread 로 자동 매핑.

## 각 annotation 별 함정

### @Async

```java
@Async
public CompletableFuture<String> fetch() { ... }    // OK — public, 다른 bean 에서 호출
@Async
private void privateAsync() { ... }                 // ✗ private 무시
@Async
String returnString() { ... }                       // ✗ Future/CompletableFuture 가 아닌 반환은 fire-and-forget
```

- public 메서드만 advice 적용
- 반환은 `void` / `Future<T>` / `CompletableFuture<T>` 만 의미

### @Cacheable

```java
@Cacheable("users")
public User find(Long id) { ... }                   // OK
public User findRecursive(Long id) {
    return find(id);                                // ✗ 첫 호출은 cache, recursive call 은 self → cache miss
}
```

- `key` SpEL 정확히 (`#id` 등)
- `condition` / `unless` 로 캐싱 조건 명시

### @Transactional

[[transaction-requires-new-ignored]] 참조.

## 진단

```java
// runtime 에 bean 이 proxy 인지 확인
@Autowired ApplicationContext ctx;

var bean = ctx.getBean(NotificationService.class);
System.out.println(AopUtils.isAopProxy(bean));        // true
System.out.println(AopUtils.isJdkDynamicProxy(bean)); // interface proxy 여부
System.out.println(AopUtils.isCglibProxy(bean));      // class proxy 여부
System.out.println(bean.getClass().getName());        // ...$$EnhancerByCGLIB$$...
```

## 재발 방지

- **ArchUnit 룰**: self-invocation 의 AOP annotation 호출 차단
  ```java
  @Test
  void shouldNotSelfInvokeAsyncMethods() {
      noMethods()
        .should().callMethodWhere(
            target("@Async").and(declaredIn(self())));
  }
  ```
- **PR 검토 체크리스트**: 새 AOP annotation 추가 시 (a) public 인가 (b) 같은 클래스 안에서 호출되지 않는가
- **테스트**: `@SpyBean` 으로 호출 횟수 검증
  ```java
  @SpyBean EmailSender sender;

  @Test
  void shouldCallSenderAsync() {
      service.send(user);
      verify(sender, timeout(1000).times(1)).send(user);
  }
  ```

## 관련

- [[backend]]
- [[transaction-requires-new-ignored]] — 같은 원인의 트랜잭션 케이스
- [[circular-dependency]] — proxy 가 circular 해결에 영향
- [[../../20-areas/backend/spring/spring|20-areas/spring]]
