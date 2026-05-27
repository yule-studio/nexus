---
title: "09. Logging — Logback + MDC + X-Request-Id"
kind: guide
status: current
created_at: 2026-05-27T11:40:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, kotlin-spring, guidelines, logging, logback, mdc]
---

# 09. Logging — Logback + MDC + X-Request-Id

**[[guidelines|↑ Hub]]** ← [[08-actuator-monitoring]] · → 다음 [[10-devtools-hotreload]]

> Spring Boot 의 logging 표준 = **Logback** (SLF4J facade). 운영에선 **JSON log + MDC + request id 추적** 이 기본.

---

## 1. Logger 사용 — Kotlin

```kotlin
import org.slf4j.Logger
import org.slf4j.LoggerFactory


@Service
class MemoService {

    private val log: Logger = LoggerFactory.getLogger(javaClass)

    fun create(...): Memo {
        log.info("create memo: title={}", title)            // INFO
        log.debug("payload: {}", req)                         // DEBUG
        try {
            // ...
        } catch (e: Exception) {
            log.error("memo create failed", e)                // 마지막 인자가 Throwable
            throw e
        }
        return memo
    }
}
```

### 1.1 companion object 로 logger 분리

```kotlin
@Service
class MemoService {
    companion object {
        private val log = LoggerFactory.getLogger(MemoService::class.java)
    }
    // ...
}
```

→ instance 별이 아니라 class 단위. 약간 성능 우위.

### 1.2 `{}` placeholder 의 lazy evaluation

```kotlin
// ❌ string interpolation — log level off 여도 evaluation
log.debug("user = ${user.toString()}")

// ✅ placeholder — DEBUG off 면 evaluation 안 함
log.debug("user = {}", user)
```

> 매 요청 수천 번 호출되는 코드 = 차이 큼.

### 1.3 `kotlin-logging` (선택 — 더 idiom)

```bash
implementation("io.github.oshai:kotlin-logging-jvm:6.0.9")
```

```kotlin
import io.github.oshai.kotlinlogging.KotlinLogging

private val log = KotlinLogging.logger {}                    // top-level — class 자동 감지

@Service
class MemoService {
    fun create(...): Memo {
        log.info { "create memo: title=$title" }              // lambda — lazy
        // ...
    }
}
```

→ `log.info { ... }` lambda — level off 시 string 생성 안 함. SLF4J `{}` 와 같은 성능.

---

## 2. application.yml — 기본 logging 설정

```yaml
# application.yml
logging:
  level:
    root: INFO
    com.example.memo: DEBUG                                  # 우리 패키지는 DEBUG
    org.springframework.security: INFO
    org.hibernate.SQL: DEBUG                                 # SQL log (dev/staging)
    org.hibernate.orm.jdbc.bind: TRACE                       # bind parameter (dev only)
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level [%X{requestId:-}] %logger{36} - %msg%n"
```

> **`[%X{requestId:-}]`** = MDC 의 requestId. 없으면 `-`. 다음 §3 참고.

### 2.1 환경별 분리

```yaml
# application-local.yml
logging:
  level:
    com.example.memo: DEBUG
    org.hibernate.SQL: DEBUG

# application-prod.yml
logging:
  level:
    com.example.memo: INFO
    org.hibernate.SQL: WARN
  pattern:
    console: ""                                               # JSON log 쓸 때는 pattern 비움
```

---

## 3. MDC (Mapped Diagnostic Context) — 한 요청 추적

**MDC** = 한 thread (= 한 request) 의 context 저장소. 모든 log 에 자동으로 attribute 포함.

```
[req-abc1234] INFO  c.e.m.MemoController - GET /api/v1/memos
[req-abc1234] INFO  c.e.m.MemoService - listByOwner: userId=1
[req-abc1234] DEBUG c.e.m.MemoRepository - findByOwnerId: ownerId=1
[req-abc1234] INFO  c.e.m.MemoController - response: 200, count=5
```

→ `req-abc1234` 로 grep 하면 한 요청의 전체 흐름 확인.

### 3.1 RequestIdFilter

```kotlin
// src/main/kotlin/com/example/memo/common/filter/RequestIdFilter.kt
package com.example.memo.common.filter

import jakarta.servlet.FilterChain
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.slf4j.MDC
import org.springframework.core.Ordered
import org.springframework.core.annotation.Order
import org.springframework.stereotype.Component
import org.springframework.web.filter.OncePerRequestFilter
import java.util.UUID

@Component
@Order(Ordered.HIGHEST_PRECEDENCE)                            // JWT filter 보다도 먼저
class RequestIdFilter : OncePerRequestFilter() {

    companion object {
        const val HEADER = "X-Request-Id"
        const val MDC_KEY = "requestId"
    }

    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain,
    ) {
        val incoming = request.getHeader(HEADER)
        val requestId = if (incoming.isNullOrBlank()) "req-${UUID.randomUUID().toString().substring(0, 8)}"
                        else incoming

        MDC.put(MDC_KEY, requestId)
        response.setHeader(HEADER, requestId)                  // 클라가 응답에서도 확인 가능
        try {
            filterChain.doFilter(request, response)
        } finally {
            MDC.remove(MDC_KEY)                                // ⚠️ 필수 — thread pool 재사용 시 누수
        }
    }
}
```

> **`MDC.remove()` 누락 함정**: thread pool 이 다음 요청에 같은 thread 재사용 → 이전 requestId 가 남음. **`finally` 안 remove 강제**.

### 3.2 user id 도 MDC 에

```kotlin
// JwtAuthenticationFilter 의 SecurityContextHolder 설정 후
MDC.put("userId", user.id.toString())
```

→ log pattern 에 `%X{userId:-}` 추가:

```yaml
logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level [req=%X{requestId:-} user=%X{userId:-}] %logger{36} - %msg%n"
```

---

## 4. JSON log — 운영 (ELK / Loki / CloudWatch)

운영에선 사람이 보는 text pattern 보다 **JSON** 이 수집 / 검색 / 알람 효율적.

### 4.1 의존성

```kotlin
// build.gradle.kts
implementation("net.logstash.logback:logstash-logback-encoder:7.4")
```

### 4.2 logback-spring.xml

```xml
<!-- src/main/resources/logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <!-- local / test — text pattern -->
    <springProfile name="local,test">
        <include resource="org/springframework/boot/logging/logback/console-appender.xml"/>
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
        </root>
        <logger name="com.example.memo" level="DEBUG"/>
        <logger name="org.hibernate.SQL" level="DEBUG"/>
    </springProfile>

    <!-- prod — JSON -->
    <springProfile name="prod,staging">
        <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder">
                <includeMdcKeyName>requestId</includeMdcKeyName>
                <includeMdcKeyName>userId</includeMdcKeyName>
                <customFields>{"app":"memo-api","env":"${SPRING_PROFILES_ACTIVE:-prod}"}</customFields>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="STDOUT"/>
        </root>
        <logger name="com.example.memo" level="INFO"/>
    </springProfile>
</configuration>
```

### 4.3 출력 예시

```json
{
  "@timestamp": "2026-05-27T01:23:45.678+00:00",
  "@version": "1",
  "message": "create memo: title=hello",
  "logger_name": "com.example.memo.MemoService",
  "thread_name": "http-nio-8080-exec-1",
  "level": "INFO",
  "level_value": 20000,
  "requestId": "req-abc1234",
  "userId": "1",
  "app": "memo-api",
  "env": "prod"
}
```

→ CloudWatch Logs Insights / ELK 에서 `requestId:"req-abc1234"` 같은 query 로 검색.

---

## 5. Access log — 모든 HTTP 요청

```yaml
# application.yml
server:
  tomcat:
    accesslog:
      enabled: true
      directory: /var/log
      prefix: access
      suffix: .log
      pattern: "%h %l %u %t \"%r\" %s %b %D"                  # %D = 응답시간 ms
      rotate: true                                              # 일 단위 rotation
```

→ Apache common log format. nginx 같이 익숙한 형식.

또는 application-level access log (envelope 응답 / 예외 정보 포함):

```kotlin
@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 10)
class AccessLogFilter : OncePerRequestFilter() {

    private val log = LoggerFactory.getLogger("access")

    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain,
    ) {
        val start = System.currentTimeMillis()
        try {
            filterChain.doFilter(request, response)
        } finally {
            val durMs = System.currentTimeMillis() - start
            log.info(
                "method={} path={} status={} dur_ms={}",
                request.method, request.requestURI, response.status, durMs,
            )
        }
    }
}
```

---

## 6. 예외 로깅 — full stack vs 요약

```kotlin
@RestControllerAdvice
class ApiExceptionHandler {

    private val log = LoggerFactory.getLogger(javaClass)

    @ExceptionHandler(BusinessException::class)
    fun handleBusiness(ex: BusinessException): ResponseEntity<ApiResponse<Nothing>> {
        if (ex.status >= 500) {
            log.error("BIZ 5xx: {}", ex.message, ex)            // ⚠️ stack trace 포함
        } else {
            log.warn("BIZ {}: {}", ex.code, ex.message)         // 4xx 는 메시지만
        }
        return ResponseEntity.status(ex.status).body(...)
    }
}
```

- **5xx** (서버 오류) = stack trace 포함 → 디버깅 필요.
- **4xx** (client 잘못) = 메시지만 → noise 줄이기.

---

## 7. SQL log — Hibernate / Hikari

### 7.1 Hibernate

```yaml
logging:
  level:
    org.hibernate.SQL: DEBUG                                  # SQL 자체
    org.hibernate.orm.jdbc.bind: TRACE                        # bind parameter (?)
```

```
DEBUG o.h.SQL - select user0_.id from users user0_ where user0_.email=?
TRACE o.h.o.j.bind - binding parameter (1:VARCHAR) <- [yuchan@example.com]
```

→ 운영은 `WARN` 또는 `OFF` (성능). dev / staging 만 DEBUG.

### 7.2 p6spy — pretty SQL log

```kotlin
implementation("com.github.gavlyukovskiy:p6spy-spring-boot-starter:1.9.0")
```

→ default 로 SQL 을 한 줄로 깔끔하게 (bind param 자동 inline). dev/staging only.

---

## 8. Sentry / 에러 추적 (옵션)

```kotlin
// build.gradle.kts
implementation("io.sentry:sentry-spring-boot-starter-jakarta:7.10.0")
```

```yaml
# application.yml
sentry:
  dsn: ${SENTRY_DSN:}
  traces-sample-rate: 0.1                                     # 10% trace sampling
  environment: ${SPRING_PROFILES_ACTIVE:local}
  release: "@project.version@"
```

→ ERROR 레벨 로그 자동 Sentry 전송. exception stack trace + MDC context (requestId / userId) 포함.

---

## 9. 자주 막히는 함정

### 함정 1 — `MDC.remove()` 누락
thread pool 재사용 시 이전 requestId 남음. **`finally` 안 remove 강제**.

### 함정 2 — string interpolation 사용
`log.debug("user = $user")` = DEBUG off 여도 toString() 실행. **`log.debug("user = {}", user)`** 또는 `kotlin-logging` lambda.

### 함정 3 — logback-spring.xml 파일명
**`logback.xml`** 아니라 **`logback-spring.xml`** — Spring profile / 환경변수 인식.

### 함정 4 — JSON log + Docker 의 multi-line stack trace
default 로 stack trace 가 multi-line — Docker / CloudWatch 가 별도 로그로 인식. **`logstash-logback-encoder` 의 `stackTraceAsArray` 옵션** 또는 single-line encoder.

### 함정 5 — log pattern 의 `%X{key:-}` syntax
default 값 표시 — `:-` 다음에 default. `%X{requestId:-N/A}` 면 `N/A`.

### 함정 6 — Kotlin `companion object` 의 logger 가 static 인줄 알고 instance method 에서 호출
`LoggerFactory.getLogger(MyClass::class.java)` 는 instance / static 무관 — class 단위. OK.

### 함정 7 — SQL log 운영
INSERT/UPDATE 마다 query + bind param = log 폭증. **운영 `WARN`**.

### 함정 8 — Sentry 가 4xx 까지 alert
`BusinessException` 4xx 도 Sentry 로 가면 알람 폭주. **`@ExceptionHandler` 안에서 5xx 만 `Sentry.captureException`** 또는 sentry config 에 ignore.

### 함정 9 — `log.error("msg", e)` 가 Kotlin 에서 ambiguous
SLF4J 의 `error(String, Object)` vs `error(String, Throwable)` overload. Kotlin 이 못 가리면 `log.error("msg", e as Throwable)` 명시.

### 함정 10 — log 안 sensitive data
비밀번호 / 토큰 / 카드번호 / PII 가 stack trace 또는 request body 에. **filter / redact** 또는 logging level 검토.

---

## 10. 다음

✅ Logging + MDC 끝.

→ [[10-devtools-hotreload]] — Spring DevTools + Hot reload + Gradle continuous build
