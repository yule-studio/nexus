---
title: "troubleshooting · Too many open files (FD leak)"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [backend, java, linux, io, resource-leak]
home_hub: "60-troubleshooting"
related:
  - "[[backend]]"
---

# Too many open files — File descriptor leak

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 — try-with-resources / lsof / ulimit |

## 증상

```
java.io.IOException: Too many open files
    at sun.nio.ch.FileChannelImpl.open0(Native Method)
    ...
java.net.SocketException: Too many open files
    at java.net.AbstractPlainSocketImpl.socketAccept(...)
```

- 시작 직후는 정상, 시간이 지나며 점진적으로 발생
- 외부 API 호출 / 파일 처리 / DB connection 모두 실패
- `lsof -p <pid> | wc -l` 가 1024 (default ulimit) 또는 65535 도달
- 컨테이너 환경: 더 빨리 도달 (ulimit 작음)

## 가설

- `ulimit -n` 늘리면 해결? → 시간만 끌고 결국 다시 도달
- GC 가 close 해줄 거 아닌가? → finalizer 의존 — 시점 보장 X, GC 안 돌면 누적
- 실제: stream / socket / connection 을 명시적으로 close 안 한 경로

## 원인 (root cause)

JVM 의 모든 `InputStream` / `OutputStream` / `Channel` / `Socket` / `Connection` 은 OS file descriptor 1개 점유. close() 안 호출 시 GC 까지 살아있음. GC 가 finalize 호출하면 close 되지만:

- young GC 만 돌면 long-lived FD 미회수
- ZGC / G1 가 promotion 늦으면 GC 사이클 사이에 FD 폭증
- finalizer 가 deprecated (Java 9+) — Cleaner 권장

```java
// Bad
InputStream is = url.openStream();
String content = new String(is.readAllBytes());   // 예외 시 close 안 됨
return content;

FileInputStream fis = new FileInputStream(path);
byte[] data = fis.readAllBytes();
// fis.close() 까먹음
```

## 해결

### 즉시 (workaround)

```bash
# ulimit 임시 확대
ulimit -n 65535

# /etc/security/limits.conf 영구
*  soft  nofile  65535
*  hard  nofile  65535

# systemd unit
[Service]
LimitNOFILE=65535

# Docker
docker run --ulimit nofile=65535:65535 ...

# K8s — 컨테이너 ulimit 직접 못 바꿈, init container 또는 securityContext
```

→ 실제 leak 은 그대로 — 시간만 지연.

### 근본 (4가지)

**(1) try-with-resources** — 모든 AutoCloseable

```java
// Good
try (var is = url.openStream()) {
    return new String(is.readAllBytes());
}

try (var fis = new FileInputStream(path);
     var bis = new BufferedInputStream(fis)) {
    return bis.readAllBytes();
}

try (var stream = Files.lines(path)) {
    return stream.filter(...).count();
}
```

규칙: `new` 한 모든 Closeable 은 try-with-resources 안에. 함수 인자로 받은 건 caller 책임.

**(2) HTTP client connection pool 명시 관리**

```java
// Apache HttpClient
PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
cm.setMaxTotal(100);                // 전체 connection 한도
cm.setDefaultMaxPerRoute(20);

CloseableHttpClient client = HttpClients.custom()
    .setConnectionManager(cm)
    .setConnectionTimeToLive(60, TimeUnit.SECONDS)   // 유효 시간
    .evictIdleConnections(30, TimeUnit.SECONDS)      // idle 정리
    .build();

// 사용 후
try (CloseableHttpResponse resp = client.execute(get)) {
    // ...
}   // ← resp close → connection pool 반환
```

```java
// Spring 5+ WebClient (reactor-netty)
@Bean
public WebClient webClient() {
    return WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(
            HttpClient.create(ConnectionProvider.builder("custom")
                .maxConnections(100)
                .maxIdleTime(Duration.ofSeconds(30))
                .maxLifeTime(Duration.ofMinutes(5))
                .build())))
        .build();
}
```

**(3) Spring RestTemplate — request factory 명시**

```java
@Bean
public RestTemplate restTemplate() {
    HttpComponentsClientHttpRequestFactory f = new HttpComponentsClientHttpRequestFactory(
        HttpClients.custom()
            .setConnectionManager(connectionManager())
            .build()
    );
    f.setConnectTimeout(2000);
    f.setReadTimeout(5000);
    return new RestTemplate(f);
}
```

→ default 는 매 호출 새 connection (close 안 하면 leak). pool 사용 강제.

**(4) JPA Stream / ResultSet 강제 close**

```java
@Transactional(readOnly = true)
public void process() {
    try (Stream<Order> stream = orderRepo.streamAll()) {
        stream.forEach(this::handle);
    }   // ← stream close → ResultSet / Statement close
}
```

## 진단

```bash
# Java 프로세스의 FD 사용량
lsof -p <pid> | wc -l                                    # 총 개수
lsof -p <pid> | awk '{print $5}' | sort | uniq -c | sort -rn   # 타입별 (REG/sock/...)
lsof -p <pid> | grep -E 'sock|pipe' | wc -l              # socket / pipe

# 시간 추적 — 1초 마다
watch -n 1 'lsof -p <pid> | wc -l'

# Limit 확인
cat /proc/<pid>/limits | grep "Max open files"

# 가장 많이 열린 file/host
lsof -p <pid> | awk '{print $9}' | sort | uniq -c | sort -rn | head -20
```

JVM 내부:
```java
// JMX MBean
ManagementFactory.getOperatingSystemMXBean();
// UnixOperatingSystemMXBean.getOpenFileDescriptorCount()

// Micrometer
process_files_open_files
process_files_max_files
```

## 재발 방지

- **SpotBugs / ErrorProne**: AutoCloseable close 누락 검출
- **PR 룰**: `new FileInputStream` / `URL.openStream` / `HttpClient` 호출 시 try-with-resources 의무
- **알람**:
  ```
  - alert: HighFileDescriptor
    expr: process_files_open_files / process_files_max_files > 0.8
    for: 5m
  ```
- **부하 테스트 후 FD 확인**: load test 후 `lsof | wc -l` 가 baseline 으로 복귀하는지

## 관련

- [[backend]]
- [[hikari-connection-leak]] — DB connection 도 FD 1개
- [[thread-pool-exhausted]] — socket FD 폭증과 연관
