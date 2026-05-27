---
title: "troubleshooting · Thread pool 고갈 — request 가 멈춤"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [backend, spring, tomcat, thread, async, virtual-thread]
home_hub: "60-troubleshooting"
related:
  - "[[backend]]"
  - "[[hikari-connection-leak]]"
---

# Thread pool 고갈 — request 가 멈춤

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 — tomcat / async / virtual thread |

## 증상

- 평소 정상, 외부 API 지연 시 모든 endpoint 가 느려짐 → 504
- thread dump:
  ```
  "http-nio-8080-exec-1" RUNNABLE
    at sun.nio.ch.SocketChannelImpl.read (network 대기)
    at org.apache.http.impl.io.SessionInputBufferImpl.read
    ...
  "http-nio-8080-exec-2" RUNNABLE  (같은 stack)
  "http-nio-8080-exec-200" RUNNABLE  (같은 stack)
  ```
- 새 request 가 queue 에 쌓이다 connection refused / timeout

## 가설

- Tomcat thread max=200 인데도 부족? → 늘려도 결국 같은 외부 호출에 막힘
- CPU 가 부족? → CPU 30% 인데도 처리량 0
- 실제: blocking IO 가 worker thread 점유 → 한 외부 API 가 모든 thread 잠금

## 원인 (root cause)

Spring MVC + Tomcat 의 thread model:

```
NIO connector → accept → worker thread pool (max=200)
                          ↓
                          [handler 호출 → 끝까지 thread 점유]
```

handler 안에서 외부 API 호출 (3초 대기) → 그 동안 worker thread 1개 점유. 동시 외부 호출 200 → pool 100% 점유 → 다른 request 가 처리 못함.

```java
// Bad — blocking IO + Tomcat worker thread
@GetMapping("/orders/{id}")
public Order get(@PathVariable Long id) {
    Order o = repo.findById(id);
    // 외부 API 3초
    ExternalDto d = restTemplate.exchange(...).getBody();
    o.enrich(d);
    return o;
}
```

## 해결

### 즉시 (workaround)

```yaml
server:
  tomcat:
    threads:
      max: 400         # 임시 확대 (메모리 영향 검토)
      min-spare: 50
    accept-count: 200
```

→ 시간만 끌고 결국 같은 패턴이면 다시 고갈.

### 근본 (4가지)

**(1) Virtual thread (Java 21 + Spring 6.1+)** — 가장 단순

```yaml
spring:
  threads:
    virtual:
      enabled: true   # ← 한 줄
```

- Tomcat worker / `@Async` / `RestClient` 가 모두 virtual thread 로 자동 전환
- blocking IO 가 OS thread 점유 X → 사실상 무한 동시성 (메모리 한계까지)
- 기존 코드 거의 그대로
- 주의: synchronized block 안 blocking IO 는 carrier thread pinning → `ReentrantLock` 으로 변경

**(2) WebFlux / Reactive** — 큰 변경

```java
@GetMapping("/orders/{id}")
public Mono<Order> get(@PathVariable Long id) {
    return orderRepo.findById(id)                       // R2DBC
        .flatMap(o -> webClient.get().retrieve()
            .bodyToMono(ExternalDto.class)
            .map(o::enrich));
}
```

장점: event loop 모델 → 적은 thread 로 큰 동시성. 단점: 전체 stack reactive (R2DBC / WebFlux / WebClient).

**(3) `@Async` + 별도 pool 분리** — 외부 호출만 격리

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean(name = "externalApiExecutor")
    public Executor externalApiExecutor() {
        ThreadPoolTaskExecutor e = new ThreadPoolTaskExecutor();
        e.setCorePoolSize(50);
        e.setMaxPoolSize(100);
        e.setQueueCapacity(200);
        e.setThreadNamePrefix("ext-api-");
        e.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        return e;
    }
}

@Service
public class OrderService {
    @Async("externalApiExecutor")
    public CompletableFuture<ExternalDto> fetchExternal(Long id) {
        return CompletableFuture.completedFuture(restTemplate.exchange(...).getBody());
    }
}

// Controller
public CompletableFuture<Order> get(@PathVariable Long id) {
    return service.fetchExternal(id)
        .thenApply(d -> orderRepo.findById(id).orElseThrow().enrich(d));
}
```

Tomcat worker 는 빠르게 free → 외부 호출은 분리 pool 에서.

**(4) Circuit breaker + timeout** — cascading 차단

```java
@Service
public class ExternalApi {
    @CircuitBreaker(name = "external", fallbackMethod = "fallback")
    @TimeLimiter(name = "external")
    public CompletableFuture<ExternalDto> fetch(Long id) {
        return CompletableFuture.supplyAsync(() ->
            restTemplate.exchange(...).getBody());
    }

    public CompletableFuture<ExternalDto> fallback(Long id, Throwable t) {
        return CompletableFuture.completedFuture(ExternalDto.empty());
    }
}
```

```yaml
resilience4j.circuitbreaker.instances.external:
  slidingWindowSize: 100
  failureRateThreshold: 50
  waitDurationInOpenState: 30s
resilience4j.timelimiter.instances.external:
  timeoutDuration: 2s
```

## RestTemplate / WebClient timeout 명시

기본값은 무한 대기 (= 위험).

```java
@Bean
public RestTemplate restTemplate() {
    HttpComponentsClientHttpRequestFactory f = new HttpComponentsClientHttpRequestFactory();
    f.setConnectTimeout(2000);
    f.setReadTimeout(5000);
    return new RestTemplate(f);
}

// 또는 Spring 6.1+ RestClient
@Bean
public RestClient restClient() {
    return RestClient.builder()
        .requestFactory(new HttpComponentsClientHttpRequestFactory(/* timeout */))
        .build();
}
```

## 진단

```bash
# Thread dump
jstack <pid> > thread.txt
grep "http-nio" thread.txt | head -30
grep "java.net.SocketInputStream.socketRead0" thread.txt | wc -l   # blocking IO count

# Micrometer (Spring Actuator)
tomcat_threads_busy_threads
tomcat_threads_current_threads
http_server_requests_seconds_count{outcome="SERVER_ERROR"}

# Async pool
executor_active_threads{name="externalApiExecutor"}
executor_queue_remaining_tasks{name="externalApiExecutor"}

# Virtual thread (Java 21)
jvm_threads_states{state="runnable"}
```

## 재발 방지

- **모든 외부 호출에 timeout 명시** — connect/read 둘 다
- **PR 룰**: `RestTemplate` / `WebClient` 호출 추가 시 timeout 검토 의무
- **부하 테스트**: 외부 API 지연 (toxiproxy / WireMock 1초+ delay) 시 자체 SLO 영향
- **알람**:
  ```
  - alert: TomcatThreadHigh
    expr: tomcat_threads_busy_threads / tomcat_threads_config_max > 0.8
    for: 2m
  ```
- Spring 6.1 + Java 21 환경이면 **virtual thread enable** 검토 — 작은 변경으로 큰 효과

## 관련

- [[backend]]
- [[hikari-connection-leak]] — connection pool ↔ thread pool cascading
- [[oom-heap]] — thread 폭증 시 heap 영향
- 책: Java Concurrency in Practice
