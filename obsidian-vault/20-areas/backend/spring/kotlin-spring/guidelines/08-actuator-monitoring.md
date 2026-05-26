---
title: "08. Spring Boot Actuator + Prometheus 모니터링"
kind: guide
status: current
created_at: 2026-05-27T11:25:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, kotlin-spring, guidelines, actuator, monitoring, prometheus]
---

# 08. Spring Boot Actuator + Prometheus 모니터링

**[[guidelines|↑ Hub]]** ← [[07-controller-patterns]] · → 다음 [[09-logging-mdc]]

> **Spring Boot Actuator** = 운영용 endpoint 묶음 (health / metrics / info / env / heapdump 등).
> Prometheus / Grafana / ELK 와 통합 — Kubernetes liveness / readiness probe 도.

---

## 1. Actuator 가 제공하는 것

| Endpoint | 용도 | 예시 응답 |
| --- | --- | --- |
| `/actuator/health` | 헬스체크 | `{"status":"UP"}` |
| `/actuator/info` | 빌드 정보 | git commit / build time |
| `/actuator/metrics` | 메트릭 목록 | jvm.memory.used / http.server.requests / ... |
| `/actuator/metrics/{name}` | 특정 메트릭 | `?tag=status:200` 으로 filter |
| `/actuator/prometheus` | Prometheus scrape | text format |
| `/actuator/env` | 환경변수 | (운영 노출 X — 비밀 노출) |
| `/actuator/loggers` | 로그 레벨 조회/변경 | runtime 에 level 변경 |
| `/actuator/heapdump` | JVM heap dump | OOM 디버깅 |
| `/actuator/threaddump` | thread dump | deadlock 추적 |
| `/actuator/mappings` | URL 매핑 목록 | controller / endpoint 확인 |
| `/actuator/beans` | Spring bean 목록 | DI 디버깅 |
| `/actuator/scheduledtasks` | `@Scheduled` 목록 | |

→ 운영엔 보통 **health / info / prometheus** 만 노출. 나머지는 admin only 또는 비활성.

---

## 2. 의존성

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    runtimeOnly("io.micrometer:micrometer-registry-prometheus")        // /actuator/prometheus
}
```

---

## 3. application.yml — 노출 endpoint 명시

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics,loggers       # 명시
        # exclude: env,heapdump,threaddump                    # default 비활성
      base-path: /actuator                                     # default
  endpoint:
    health:
      show-details: when-authorized                            # never / always / when-authorized
      probes:
        enabled: true                                          # liveness / readiness 분리
    info:
      enabled: true
    prometheus:
      enabled: true

  metrics:
    tags:
      application: memo-api                                    # 모든 metric 에 tag 추가
      env: ${SPRING_PROFILES_ACTIVE:local}

  info:
    git:
      mode: full                                               # build/info 의 git 정보 노출
    env:
      enabled: true

# Git info — info endpoint 에 표시
info:
  app:
    name: memo-api
    version: "@project.version@"
```

### 3.1 운영 vs dev

```yaml
# application-prod.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus                        # admin endpoint 제외
  endpoint:
    health:
      show-details: never                                      # 내부 정보 노출 X
```

```yaml
# application-local.yml
management:
  endpoints:
    web:
      exposure:
        include: "*"                                            # dev 는 모두 노출
  endpoint:
    health:
      show-details: always
```

---

## 4. Health endpoint — 상세

### 4.1 기본

```bash
curl http://localhost:8080/actuator/health
# {"status":"UP"}

# show-details: always 면
curl http://localhost:8080/actuator/health
# {
#   "status": "UP",
#   "components": {
#     "db": { "status": "UP", "details": { "database": "PostgreSQL", "validationQuery": "isValid()" } },
#     "diskSpace": { "status": "UP", "details": { "total": 500000000000, "free": ... } },
#     "ping": { "status": "UP" }
#   }
# }
```

### 4.2 Kubernetes liveness / readiness 분리

`management.endpoint.health.probes.enabled: true` 면 자동 생성:

- `/actuator/health/liveness` — 죽었나? (재시작 필요) → 보통 항상 UP, deadlock 시 DOWN
- `/actuator/health/readiness` — 받을 준비? (트래픽 라우팅) → DB / external 의존 포함

```yaml
# Kubernetes manifest
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

### 4.3 custom HealthIndicator

외부 의존 (Redis / S3 / 외부 API) 도 체크:

```kotlin
// src/main/kotlin/com/example/memo/health/RedisHealthIndicator.kt
package com.example.memo.health

import org.springframework.boot.actuate.health.Health
import org.springframework.boot.actuate.health.HealthIndicator
import org.springframework.data.redis.core.StringRedisTemplate
import org.springframework.stereotype.Component

@Component
class RedisHealthIndicator(
    private val redis: StringRedisTemplate,
) : HealthIndicator {

    override fun health(): Health = try {
        val pong = redis.connectionFactory?.connection?.use { it.ping() }
        if (pong == "PONG") Health.up().withDetail("ping", "PONG").build()
        else Health.down().withDetail("ping", pong ?: "null").build()
    } catch (e: Exception) {
        Health.down(e).build()
    }
}
```

→ `/actuator/health` 의 components 에 `redis` 자동 추가.

---

## 5. Info endpoint

빌드 시점 git / build 정보 표시:

```kotlin
// build.gradle.kts
plugins {
    // ...
    id("com.gorylenko.gradle-git-properties") version "2.4.1"     // git.properties 자동 생성
}
```

```bash
./gradlew bootRun
curl http://localhost:8080/actuator/info
# {
#   "app": { "name": "memo-api", "version": "0.0.1-SNAPSHOT" },
#   "git": {
#     "branch": "main",
#     "commit": { "id": "abc1234", "time": "2026-05-27T..." }
#   }
# }
```

> 운영에서 어느 commit 이 배포됐는지 확인할 때 유용.

---

## 6. Prometheus 통합

### 6.1 scrape endpoint

```bash
curl http://localhost:8080/actuator/prometheus
# # HELP jvm_memory_used_bytes The amount of used memory
# # TYPE jvm_memory_used_bytes gauge
# jvm_memory_used_bytes{area="heap",id="G1 Eden Space",} 1.048576E8
# ...
# http_server_requests_seconds_count{method="GET",status="200",uri="/api/v1/memos",} 142.0
# http_server_requests_seconds_sum{method="GET",status="200",uri="/api/v1/memos",} 0.842
```

### 6.2 Prometheus config

```yaml
# prometheus.yml
scrape_configs:
  - job_name: memo-api
    metrics_path: /actuator/prometheus
    static_configs:
      - targets: ["memo-api:8080"]
    scrape_interval: 15s
```

### 6.3 자주 보는 메트릭

| metric | 의미 |
| --- | --- |
| `http_server_requests_seconds_*` | endpoint 별 요청 수 / 지연 (label: method, status, uri) |
| `jvm_memory_used_bytes` | heap / non-heap 사용량 |
| `jvm_threads_states_threads` | thread 상태 (RUNNABLE / BLOCKED / WAITING) |
| `jvm_gc_pause_seconds_*` | GC 시간 |
| `hikaricp_connections_active` | DB pool active connection |
| `hikaricp_connections_pending` | DB pool 대기 (높으면 pool size 부족) |
| `process_cpu_usage` | CPU 사용율 |
| `tomcat_threads_busy_threads` | servlet thread busy |

### 6.4 custom metric — Micrometer

```kotlin
import io.micrometer.core.instrument.Counter
import io.micrometer.core.instrument.MeterRegistry
import org.springframework.stereotype.Service

@Service
class MemoService(
    private val memoRepository: MemoRepository,
    private val meterRegistry: MeterRegistry,
) {
    private val createdCounter: Counter = Counter
        .builder("memo.created")
        .description("Memo 생성 횟수")
        .tag("env", "prod")
        .register(meterRegistry)

    fun create(owner: User, title: String, content: String): Memo {
        val memo = Memo(owner = owner, title = title, content = content)
        memoRepository.save(memo)
        createdCounter.increment()
        return memo
    }
}
```

→ `/actuator/prometheus` 에 `memo_created_total{env="prod"}` 자동 노출.

### 6.5 Timer — 메서드 실행 시간

```kotlin
import io.micrometer.core.annotation.Timed

@Service
class SlowService {
    @Timed(value = "slow.operation", description = "느린 작업 시간")
    fun doIt() {
        // ...
    }
}
```

→ `slow_operation_seconds_count` / `slow_operation_seconds_sum` 자동.

> `@Timed` 동작 위해 `aspectjweaver` 의존성 + Spring AOP 활성.

---

## 7. Logger endpoint — runtime 로그 레벨 변경

```bash
# 현재 레벨 조회
curl http://localhost:8080/actuator/loggers/com.example.memo
# {"configuredLevel":null,"effectiveLevel":"INFO"}

# 변경 — DEBUG
curl -X POST http://localhost:8080/actuator/loggers/com.example.memo \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel":"DEBUG"}'

# 검증
curl http://localhost:8080/actuator/loggers/com.example.memo
# {"configuredLevel":"DEBUG","effectiveLevel":"DEBUG"}
```

→ 운영 중 디버깅 시 재배포 없이 로그 레벨 변경.

---

## 8. SecurityConfig — actuator 경로 권한

```kotlin
// SecurityConfig 의 authorizeHttpRequests
authorize("/actuator/health", permitAll)               // 누구나 (LB / k8s probe)
authorize("/actuator/health/liveness", permitAll)
authorize("/actuator/health/readiness", permitAll)
authorize("/actuator/info", permitAll)
authorize("/actuator/prometheus", permitAll)            // 내부 망 only (firewall)
authorize("/actuator/**", hasRole("ADMIN"))             // 나머지는 admin 만
```

또는 운영 환경에서 actuator 만 다른 port:

```yaml
management:
  server:
    port: 8081                                          # 별도 port — LB 가 외부에 노출 안 함
```

---

## 9. Grafana 대시보드 — 추천

- **JVM (Micrometer) by Apache** — JVM 메모리 / GC / threads
- **Spring Boot Statistics** — HTTP 요청 / endpoint 별 latency
- **PostgreSQL Database** — Hikari pool / DB 메트릭

[Grafana Dashboards](https://grafana.com/grafana/dashboards/) 에서 검색 후 import.

---

## 10. 함정 모음

### 함정 1 — `expose: "*"` 운영
모든 endpoint 노출 = `/actuator/env` 가 환경변수 (비밀 포함) 노출. **`health,info,prometheus`** 만.

### 함정 2 — `show-details: always` 운영
DB driver / pool 상세 노출. **운영 `never`**.

### 함정 3 — `/actuator/prometheus` 가 외부 노출
누구나 metric 수집 = APM 우회. **내부 망 only** (firewall / VPN).

### 함정 4 — Prometheus scrape 가 너무 잦음
1초마다 scrape = 부하. **15s ~ 60s**.

### 함정 5 — custom HealthIndicator 가 느림
health check 가 5s+ 걸리면 k8s probe 가 fail. **timeout 명시 + 가벼운 ping**.

### 함정 6 — `hikaricp_connections_pending` 가 0 이상
pool size 부족 = 요청 대기. **pool size ↑** (단 DB max_connections 한도 내).

### 함정 7 — `@Timed` 어노테이션 동작 X
`aspectjweaver` 의존성 + `@EnableAspectJAutoProxy`. **build.gradle.kts 확인**.

### 함정 8 — k8s readiness probe 가 startup 시 fail
앱이 너무 늦게 준비. `initialDelaySeconds` 충분히 (30s+).

---

## 11. 다음

✅ Actuator + Prometheus 끝.

→ [[09-logging-mdc]] — Logback / MDC / X-Request-Id 추적
