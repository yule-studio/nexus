---
title: "troubleshooting · OutOfMemoryError: Java heap space"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [backend, jvm, gc, memory, oom, spring]
home_hub: "60-troubleshooting"
related:
  - "[[backend]]"
  - "[[oom-metaspace]]"
---

# OutOfMemoryError: Java heap space

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 — heap dump 분석 / streaming / pagination 정리 |

## 증상

```
java.lang.OutOfMemoryError: Java heap space
    at java.util.Arrays.copyOf(Arrays.java:3236)
    at java.util.ArrayList.grow(ArrayList.java:265)
    ...
```

- JVM 이 죽거나 container 가 OOMKilled (exit 137) → K8s 가 재시작
- GC log 에 `Full GC` 가 분 단위 반복, 처리량 0
- 특정 endpoint 호출 후 발생 (대량 조회 / 파일 처리 / 통계)
- Heap 사용량 그래프가 톱니 → 평탄 (GC 가 더 이상 회수 못함)

## 가설

- 처음에 의심: heap size 가 작음 → `-Xmx` 키워봤지만 시간만 지연
- 다음 의심: GC 알고리즘 → G1 → ZGC 시도 (latency 는 좋아지지만 OOM 은 그대로)
- 실제: 코드가 큰 결과 set 을 한번에 in-memory 로 로딩

## 원인 (root cause)

JVM heap 의 한계 ≠ 데이터 한계. 다음 패턴이 흔함:

```java
// (1) 전체 결과 로딩 후 처리
List<Order> orders = orderRepo.findAll();  // 5M rows
orders.stream().forEach(this::process);

// (2) JPA cascade fetch 폭발
Customer c = customerRepo.findById(id);  // → orders → orderItems → ... 깊은 fetch

// (3) String concat / log
String dump = list.toString();  // 큰 list 의 toString → MB 단위 String

// (4) ByteArrayOutputStream 가 큰 파일
byte[] all = stream.readAllBytes();  // 100MB 파일
```

## 해결

### 즉시 (workaround)

1. `-Xmx` 임시 증가 + 재시작 (시간 벌기)
2. 문제 endpoint 임시 차단 (feature flag) — 다른 사용자 영향 방지

### 근본 (4가지 패턴)

**(1) Stream + pagination (JPA)**

```java
// Bad
List<Order> orders = orderRepo.findAll();

// Good — pagination
int page = 0;
Page<Order> result;
do {
    result = orderRepo.findAll(PageRequest.of(page++, 1000));
    result.forEach(this::process);
} while (result.hasNext());

// Better — Stream + fetchSize (read-only)
@Transactional(readOnly = true)
Stream<Order> stream = orderRepo.streamAll();
stream.forEach(this::process);
// → @Query(...) + Stream<Order> streamAll()
//   + @QueryHints(@QueryHint(name = HINT_FETCH_SIZE, value = "1000"))
```

**(2) `EntityManager.clear()` 주기적 호출** — 1차 캐시 정리

```java
int batchSize = 1000;
List<Long> ids = ...;
for (int i = 0; i < ids.size(); i += batchSize) {
    var slice = orderRepo.findAllById(ids.subList(i, Math.min(i + batchSize, ids.size())));
    slice.forEach(this::process);
    em.flush();
    em.clear();  // ← 1차 캐시 비움
}
```

**(3) Streaming response (대용량 file / report)**

```java
@GetMapping(value = "/export.csv", produces = "text/csv")
public ResponseEntity<StreamingResponseBody> export() {
    StreamingResponseBody body = out -> {
        try (var writer = new PrintWriter(out);
             var stream = orderRepo.streamAll()) {
            stream.forEach(o -> writer.println(toCsv(o)));
        }
    };
    return ResponseEntity.ok().body(body);
}
```

**(4) JVM 옵션** (코드 수정 못할 때 일시 완화)

```
-Xms2g -Xmx2g                     # heap 고정
-XX:+UseG1GC                       # 또는 -XX:+UseZGC (Java 21)
-XX:MaxRAMPercentage=75            # container 기준 비율
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/app/heap.hprof
-XX:+ExitOnOutOfMemoryError
```

## Heap dump 분석

```bash
# OOM 시 자동 생성된 hprof
jmap -dump:format=b,file=heap.bin <pid>          # 수동 dump
jhat heap.bin                                    # 가벼운 분석

# 또는 Eclipse MAT (권장)
# - Leak Suspects Report → 의심 객체 / retained heap top
# - Dominator Tree → 누가 메모리 가장 많이 보유
# - Histogram → 클래스별 instance count / size
```

자주 발견되는 leak suspect:
- `HashMap$Node[]` 큰 사이즈 → cache 가 unbounded
- `String` / `char[]` → log / DTO toString
- `org.hibernate.engine.spi.ActionQueue` → 트랜잭션 안에 누적된 entity (flush + clear 필요)
- `SimpleDateFormat` ThreadLocal → ThreadLocal cleanup 누락

## 재발 방지

- **Container 운영**: `-XX:MaxRAMPercentage=75` + K8s `memory.limit` + heap dump volume mount
- **모니터링**: Micrometer + Prometheus 로 `jvm_memory_used_bytes` / `jvm_gc_pause_seconds` 알람
  ```
  - alert: HeapHigh
    expr: jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} > 0.85
    for: 5m
  ```
- **부하 테스트**: 새 endpoint 추가 시 k6 / Gatling 으로 sustained load → heap 곡선 확인
- **코드 룰**: `findAll()` / `readAllBytes()` 등 unbounded API 호출 시 lint warning

## 관련

- [[backend]]
- [[oom-metaspace]] — Metaspace OOM (다른 영역)
- [[hikari-connection-leak]] — connection leak 도 heap 영향
- [[../../20-areas/backend/jvm/jvm|20-areas/jvm]]
