---
title: "02. 프로젝트 구조 + application.yml + Config"
kind: guide
status: current
created_at: 2026-05-27T10:20:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, kotlin-spring, guidelines, structure, config]
---

# 02. 프로젝트 구조 + application.yml + Config

**[[guidelines|↑ Hub]]** ← [[01-installation-and-hello-world]] · → 다음 [[03-domain-and-jpa]]

> 메모 API 본격 프로젝트의 패키지 구조 + application.yml + `@ConfigurationProperties`.

---

## 1. 권장 패키지 구조

```
src/main/kotlin/com/example/memo/
├── MemoApiApplication.kt           # @SpringBootApplication
├── config/                         # Spring Config (Security / Jackson / Web)
│   ├── SecurityConfig.kt
│   ├── JacksonConfig.kt
│   ├── CorsConfig.kt
│   └── AppProperties.kt            # @ConfigurationProperties
├── common/                         # 횡단 — envelope / exception / response / filter
│   ├── ApiResponse.kt
│   ├── BusinessException.kt
│   ├── ApiExceptionHandler.kt
│   └── filter/
│       └── ApiLogFilter.kt
├── domain/                         # 도메인 (헥사고날 코어)
│   ├── user/
│   │   ├── User.kt                 # JPA Entity (open class + var)
│   │   ├── UserRepository.kt       # interface (port)
│   │   └── UserStatus.kt           # enum
│   └── memo/
│       ├── Memo.kt
│       └── MemoRepository.kt
├── infrastructure/                 # 외부 — JPA / Redis / S3 / 메일
│   ├── persistence/
│   │   └── jpa/
│   │       ├── UserJpaRepository.kt  # Spring Data JPA
│   │       └── MemoJpaRepository.kt
│   └── security/
│       ├── JwtTokenProvider.kt
│       ├── JwtAuthenticationFilter.kt
│       └── PasswordHasher.kt
├── application/                    # use case / business logic
│   ├── auth/
│   │   ├── SignupService.kt
│   │   └── LoginService.kt
│   └── memo/
│       ├── CreateMemoService.kt
│       ├── ListMemosService.kt
│       └── UpdateMemoService.kt
└── presentation/                   # REST controller + DTO
    ├── auth/
    │   ├── AuthController.kt
    │   ├── SignupRequest.kt
    │   └── LoginRequest.kt
    └── memo/
        ├── MemoController.kt
        ├── CreateMemoRequest.kt
        └── MemoResponse.kt
```

> **왜 이렇게**: 헥사고날 (Port & Adapter). domain 이 infra / Spring 모름 → 테스트 / 변경 용이.
> 작은 프로젝트는 더 단순한 구조로 시작해도 OK. **이 구조는 5–10 도메인 부터 의미 있음**.

### 1.1 더 단순한 시작 구조 (작은 프로젝트)

```
src/main/kotlin/com/example/memo/
├── MemoApiApplication.kt
├── auth/
│   ├── AuthController.kt
│   ├── AuthService.kt
│   ├── User.kt
│   └── UserRepository.kt
├── memo/
│   ├── MemoController.kt
│   ├── MemoService.kt
│   ├── Memo.kt
│   └── MemoRepository.kt
├── common/
│   └── ...
└── config/
    └── SecurityConfig.kt
```

→ 한 도메인 = 한 package. 본 가이드는 **단순 구조** 로 진행.

---

## 2. application.yml — 환경별 분리

`application.properties` 보다 `application.yml` 권장 (계층 명확).

### 2.1 기본 application.yml

```yaml
# src/main/resources/application.yml
spring:
  application:
    name: memo-api
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:memo}
    username: ${DB_USER:memo}
    password: ${DB_PASSWORD:memo}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 3000
      idle-timeout: 600000
      max-lifetime: 1800000           # PG wait_timeout 보다 짧게
  jpa:
    hibernate:
      ddl-auto: validate              # ⚠️ 운영도 validate. Flyway 가 schema 관리.
    open-in-view: false               # ⚠️ N+1 / lazy 함정 방지
    properties:
      hibernate:
        format_sql: true
        default_batch_fetch_size: 100
        order_inserts: true
        order_updates: true
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false

server:
  port: ${PORT:8080}
  servlet:
    encoding:
      charset: UTF-8

# 우리 앱 설정 (custom)
app:
  jwt:
    secret: ${JWT_SECRET:dev-jwt-secret-32-bytes-please-change-in-prod}
    access-token-expiration-minutes: 15
    refresh-token-expiration-days: 14
    issuer: memo-api
  cors:
    allowed-origins: ${CORS_ORIGINS:http://localhost:3000}

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    health:
      show-details: never            # 운영 노출 X

logging:
  level:
    org.springframework.security: INFO
    com.example.memo: DEBUG          # 우리 패키지는 DEBUG (개발)
    org.hibernate.SQL: DEBUG          # SQL log (dev/staging 만)
```

### 2.2 환경별 분리 — application-{profile}.yml

```yaml
# src/main/resources/application-local.yml — 개발 PC
spring:
  jpa:
    properties:
      hibernate:
        show_sql: true                 # console 에 SQL 출력
        format_sql: true

logging:
  level:
    com.example.memo: DEBUG
```

```yaml
# src/main/resources/application-prod.yml — 운영
spring:
  jpa:
    properties:
      hibernate:
        show_sql: false

logging:
  level:
    com.example.memo: INFO
    org.hibernate.SQL: WARN

management:
  endpoint:
    health:
      show-details: never
```

### 2.3 profile 활성화

```bash
# 환경변수
export SPRING_PROFILES_ACTIVE=local
./gradlew bootRun

# 또는 CLI argument
./gradlew bootRun --args='--spring.profiles.active=local'

# IntelliJ Run Configuration → Active Profiles: local
```

### 2.4 `.env` 같은 외부 환경변수 주입

Spring Boot 는 system env 자동 인식 — `${DB_HOST:localhost}` 처럼 default 와 함께.

`.env` 파일을 직접 읽고 싶다면 `dotenv-kotlin` 같은 라이브러리 또는 직접 export:

```bash
# .env.local (gitignore)
export DB_HOST=localhost
export DB_PASSWORD=secret
export JWT_SECRET=$(openssl rand -hex 32)

# 사용 시
source .env.local
./gradlew bootRun
```

운영에서는 Kubernetes Secret / AWS Secrets Manager / Vault 등으로 주입.

---

## 3. `@ConfigurationProperties` — 타입 안전한 설정 binding

application.yml 의 `app.jwt.*` 를 Kotlin 객체로:

```kotlin
// config/AppProperties.kt
package com.example.memo.config

import org.springframework.boot.context.properties.ConfigurationProperties

@ConfigurationProperties(prefix = "app")
data class AppProperties(
    val jwt: Jwt,
    val cors: Cors,
) {
    data class Jwt(
        val secret: String,
        val accessTokenExpirationMinutes: Long,
        val refreshTokenExpirationDays: Long,
        val issuer: String,
    )

    data class Cors(
        val allowedOrigins: List<String>,
    )
}
```

활성:

```kotlin
// MemoApiApplication.kt 또는 별도 config 에
import org.springframework.boot.context.properties.EnableConfigurationProperties

@SpringBootApplication
@EnableConfigurationProperties(AppProperties::class)
class MemoApiApplication

fun main(args: Array<String>) {
    runApplication<MemoApiApplication>(*args)
}
```

> **`data class` + `@ConfigurationProperties` (constructor binding)** — Spring Boot 3.x 권장. immutable + null-safe.

사용:

```kotlin
@Service
class JwtTokenProvider(
    private val appProperties: AppProperties,         // constructor 주입
) {
    fun generateToken(): String {
        val secret = appProperties.jwt.secret
        // ...
    }
}
```

---

## 4. Jackson 설정 — 한글 / 시간대 / data class

```kotlin
// config/JacksonConfig.kt
package com.example.memo.config

import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.databind.SerializationFeature
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule
import com.fasterxml.jackson.module.kotlin.KotlinModule
import com.fasterxml.jackson.module.kotlin.registerKotlinModule
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class JacksonConfig {

    @Bean
    fun objectMapper(): ObjectMapper = ObjectMapper().apply {
        registerKotlinModule()                          // data class default 값 / nullable 인식
        registerModule(JavaTimeModule())                // LocalDateTime / Instant 직렬화
        disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)  // ISO-8601
        disable(SerializationFeature.FAIL_ON_EMPTY_BEANS)
    }
}
```

> **`KotlinModule`** 미등록 시 data class 의 default 값 / non-null property 가 deserialization 실패. **반드시 등록**.
>
> 단 `jackson-module-kotlin` 의존성을 build.gradle.kts 에 추가했다면 (Spring Boot starter-web 이 자동 detect) Spring Boot 가 알아서 등록. 명시적 `@Bean` 은 customize 시에만.

---

## 5. CORS 설정

```kotlin
// config/CorsConfig.kt
package com.example.memo.config

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.web.cors.CorsConfiguration
import org.springframework.web.cors.UrlBasedCorsConfigurationSource

@Configuration
class CorsConfig(
    private val appProperties: AppProperties,
) {

    @Bean
    fun corsConfigurationSource(): UrlBasedCorsConfigurationSource {
        val config = CorsConfiguration().apply {
            allowCredentials = false                    // JWT + Bearer 헤더면 false 권장
            allowedOrigins = appProperties.cors.allowedOrigins
            allowedMethods = listOf("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS")
            allowedHeaders = listOf("Authorization", "Content-Type", "Accept-Language")
            maxAge = 3600L
        }
        return UrlBasedCorsConfigurationSource().apply {
            registerCorsConfiguration("/**", config)
        }
    }
}
```

> SecurityConfig 의 `cors { configurationSource = corsConfigurationSource() }` 에서 활용 (다음 [[05-security-and-deployment]]).

---

## 6. profile 별 SecurityConfig (dev vs prod)

dev 에선 모두 허용, prod 에선 인증 강제 — `@Profile` 어노테이션:

```kotlin
@Configuration
@Profile("local", "test")                              // local / test profile 만
class DevSecurityConfig {
    @Bean
    fun filterChain(http: HttpSecurity): SecurityFilterChain {
        http {
            csrf { disable() }
            authorizeHttpRequests { authorize(anyRequest, permitAll) }
        }
        return http.build()
    }
}


@Configuration
@Profile("!local & !test")                             // local / test 아닐 때
class ProdSecurityConfig {
    @Bean
    fun filterChain(http: HttpSecurity): SecurityFilterChain {
        http {
            // ... 인증 강제 (다음 노트)
        }
        return http.build()
    }
}
```

---

## 7. logback 설정 (선택)

`application.yml` 의 `logging.level` 만으로 충분하지만 더 세밀히:

```xml
<!-- src/main/resources/logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <springProfile name="local,test">
        <include resource="org/springframework/boot/logging/logback/console-appender.xml"/>
        <root level="INFO">
            <appender-ref ref="CONSOLE"/>
        </root>
        <logger name="com.example.memo" level="DEBUG"/>
        <logger name="org.hibernate.SQL" level="DEBUG"/>
    </springProfile>

    <springProfile name="prod">
        <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>     <!-- JSON log -->
        </appender>
        <root level="INFO">
            <appender-ref ref="STDOUT"/>
        </root>
    </springProfile>
</configuration>
```

> JSON log 는 ELK / CloudWatch / Loki 통합에 편함. 의존성: `net.logstash.logback:logstash-logback-encoder`.

---

## 8. Gradle 명령어 정리

```bash
./gradlew bootRun                       # 실행
./gradlew bootRun --args='--spring.profiles.active=local'

./gradlew build                          # 빌드 + 테스트
./gradlew assemble                       # 빌드만 (테스트 X)
./gradlew test                           # 테스트만

./gradlew bootJar                        # 실행 가능한 JAR 생성 → build/libs/memo-api-0.0.1-SNAPSHOT.jar

./gradlew clean                          # build 디렉터리 삭제
./gradlew dependencies                   # 의존성 트리

./gradlew --refresh-dependencies build   # 의존성 강제 재다운로드

./gradlew --stop                         # Gradle daemon 종료
```

---

## 9. 자주 막히는 함정 (구조 / 설정)

### 함정 1 — `@ConfigurationProperties` 가 binding 안 됨
`@EnableConfigurationProperties(AppProperties::class)` 누락. 또는 `data class` 가 아닌 일반 class 인데 `var` 없음.

### 함정 2 — `application.yml` 들여쓰기 깨짐
YAML 은 들여쓰기 매우 엄격. **2 spaces 일관**, tab 절대 X.

### 함정 3 — profile 활성화 안 됨
환경변수 / Run Configuration 누락. `./gradlew bootRun --args='--spring.profiles.active=local'`.

### 함정 4 — Hikari pool size 너무 작음
default 10. 운영 트래픽엔 20-30 권장. 단 PG `max_connections` 와 매칭 (workers × pool ≤ max_connections).

### 함정 5 — `JwtAuthenticationFilter` 가 `@Component` 인데 SecurityConfig 가 의존성 주입 못 받음
순환 의존성. **`@Component` 빼고 `SecurityConfig` 의 `@Bean` 으로 직접 생성**.

### 함정 6 — `JacksonConfig` 의 ObjectMapper 가 Spring Boot default 와 충돌
Boot 가 자동으로 KotlinModule 등록 — 명시적 `@Bean` 은 customize 시에만.

### 함정 7 — `.env` 파일을 import 안 했는데 환경변수 빈 채로
`export` 필요. 또는 IntelliJ EnvFile plugin.

### 함정 8 — Flyway baseline 충돌
빈 DB 에 `baseline-on-migrate: false` + V1 없음 → schema migration 시작 실패. **V1 placeholder 만들기**.

---

## 10. 다음

✅ 프로젝트 구조 + 설정 끝.

→ [[03-domain-and-jpa]] — User / Memo entity 정의 + JPA Repository + Flyway 마이그레이션
