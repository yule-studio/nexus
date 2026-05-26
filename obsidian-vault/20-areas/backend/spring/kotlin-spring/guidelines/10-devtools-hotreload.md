---
title: "10. Spring DevTools + Hot Reload + 개발 편의"
kind: guide
status: current
created_at: 2026-05-27T11:55:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, kotlin-spring, guidelines, devtools, hot-reload, live-reload]
---

# 10. Spring DevTools + Hot Reload + 개발 편의

**[[guidelines|↑ Hub]]** ← [[09-logging-mdc]]

> 코드 수정마다 손으로 재시작 = 개발 짜증. **Spring DevTools + Gradle continuous build** 로 자동 재시작 / hot reload.

---

## 1. Spring DevTools 가 하는 것

- **Auto restart** — classpath (compiled class) 변경 시 자동으로 app 재시작 (1~2초).
- **LiveReload** — 브라우저 자동 새로고침 (FE 작업 시).
- **Property defaults** — dev 친화 default (`spring.thymeleaf.cache=false` 등).
- **H2 console / remote debug** — 디버깅 편의.

> **운영 batter 자동 비활성** — `spring-boot-devtools` 가 JAR 안에 있어도 production 환경에서는 자동 disable (단 `*.original` JAR 형식 등 일부 케이스 주의).

---

## 2. 의존성

```kotlin
// build.gradle.kts
dependencies {
    developmentOnly("org.springframework.boot:spring-boot-devtools")
}
```

> **`developmentOnly`** 가 핵심 — runtime classpath 에만 포함, JAR 빌드 시 제외 (운영 안전).

---

## 3. IntelliJ 설정 — 자동 빌드

DevTools 가 동작하려면 **class 파일이 새로 생성** 되어야 함. IntelliJ default 는 수동 build.

### 3.1 Build project automatically

`Settings` → `Build, Execution, Deployment` → `Compiler` → `Build project automatically` ✅ 체크.

### 3.2 Allow auto-make to start

`Help` → `Find Action` → `Registry...` → `compiler.automake.allow.when.app.running` ✅ 체크 (IntelliJ 2021.2+ 는 default 활성).

→ 이제 코드 수정 + 저장 (`Cmd+S`) 시 자동 컴파일 → DevTools 가 재시작.

---

## 4. Gradle continuous build (대안)

IntelliJ 없이 또는 더 빠른 cycle:

```bash
./gradlew bootRun &
./gradlew build --continuous              # source 변경 감지 → 자동 compile
```

→ Gradle 이 file watch + recompile. DevTools 가 새 class 파일 인식 → 재시작.

또는 한 명령:

```bash
./gradlew bootRun -t                       # -t = continuous
```

> 단 `-t` + `bootRun` 조합은 안정성 이슈. 보통 두 터미널 분리.

---

## 5. application.yml — DevTools 옵션

```yaml
# application.yml (또는 application-local.yml)
spring:
  devtools:
    restart:
      enabled: true                         # default
      poll-interval: 1s                     # file watch 주기
      quiet-period: 1s                      # 변경 후 안정 대기
      exclude: "**/*.html,**/*.properties"  # 일부 파일은 재시작 안 함
      additional-paths: src/main/resources  # 추가 watch path
    livereload:
      enabled: true
      port: 35729                            # browser plugin port
```

---

## 6. LiveReload — 브라우저 자동 새로고침

DevTools 가 LiveReload server (port 35729) 자동 시작.

브라우저에 **LiveReload extension** 설치:
- Chrome: [LiveReload](https://chromewebstore.google.com/detail/livereload/jnihajbhpnppcggbcgedagnkighmdlei)
- Firefox: [LiveReload](https://addons.mozilla.org/firefox/addon/livereload-web-extension/)

브라우저에서 plugin 활성 → 코드 / template 수정 시 자동 새로고침.

> API 만 만드는 경우엔 의미 적음. FE template (Thymeleaf / Mustache) 작업 시 유용.

---

## 7. property override — `~/.spring-boot-devtools.properties`

dev PC 마다 다른 설정 (개인 DB 비밀번호 등):

```properties
# ~/.spring-boot-devtools.properties (gitignore 대상이지만 home 디렉터리라 자연 격리)
spring.datasource.password=my-local-db-pw
```

→ 모든 Spring Boot 프로젝트의 dev 모드에 적용. `.env` 와 비슷한 역할.

---

## 8. Lombok 대안 — 이미 Kotlin

Java 진영에서 자주 쓰는 Lombok (`@Data`, `@Builder`, `@Slf4j`) = Kotlin 에선 불필요.

| Lombok | Kotlin |
| --- | --- |
| `@Data` | `data class` |
| `@Builder` | `data class` + named arguments |
| `@Slf4j` | `private val log = LoggerFactory.getLogger(javaClass)` 또는 kotlin-logging |
| `@RequiredArgsConstructor` | primary constructor + `val` |
| `@Getter / @Setter` | `val` / `var` |
| `@NoArgsConstructor` | `kotlin("plugin.jpa")` 자동 합성 |

→ build.gradle.kts 에 lombok 의존성 추가 X.

---

## 9. JPA Buddy / Database Tools — IntelliJ Ultimate

### 9.1 JPA Buddy (free plugin)

`Settings` → `Plugins` → "JPA Buddy" 설치.

- Entity ↔ Migration (Flyway / Liquibase) 자동 diff
- Entity 생성 wizard
- DTO 자동 생성
- ER diagram

### 9.2 Database tool

IntelliJ Ultimate 의 `Database` tool window — DB 직접 연결:

`View` → `Tool Windows` → `Database` → `+` → `Data Source` → `PostgreSQL`.

→ table 보기 / query 실행 / data 편집. DBeaver 같은 별도 도구 불필요.

---

## 10. HTTP Client — IntelliJ Ultimate

Postman 대신 IntelliJ 의 `.http` 파일:

```http
### 1. signup
POST http://localhost:8080/api/v1/auth/signup
Content-Type: application/json

{"email":"yuchan@example.com","password":"P@ssw0rd1!","name":"yuchan"}

### 2. login
POST http://localhost:8080/api/v1/auth/login
Content-Type: application/json

{"email":"yuchan@example.com","password":"P@ssw0rd1!"}

> {% client.global.set("token", response.body.data.accessToken); %}

### 3. 메모 생성 — 이전 token 사용
POST http://localhost:8080/api/v1/memos
Authorization: Bearer {{token}}
Content-Type: application/json

{"title":"첫 메모","content":"안녕하세요"}
```

→ git 에 commit 가능. 팀 공유.

---

## 11. Remote debug — `java -agentlib:jdwp`

```bash
./gradlew bootRun --debug-jvm
# Listening for transport dt_socket at address: 5005
```

IntelliJ `Run → Edit Configurations → + Remote JVM Debug → port 5005` → debugger attach.

→ 운영 중 container 에 attach (보안 주의 — 절대 외부 노출 X).

---

## 12. Gradle 속도 최적화

```properties
# gradle.properties
org.gradle.daemon=true
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configureondemand=true
org.gradle.jvmargs=-Xmx4g -XX:MaxMetaspaceSize=512m
kotlin.daemon.jvmargs=-Xmx2g
```

→ build 30-50% 빨라짐.

---

## 13. Hot reload 한계 — class structure 변경

DevTools 의 auto-restart 는 **JVM 재시작** 이라 빠름 (~2s) 이지만 진짜 hot reload (재시작 없이 method body 교체) 는 아님.

**진짜 hot reload** 가 필요하면:
- **JRebel** (유료) — class structure 변경도 hot reload
- **HotSwap Agent** (오픈소스) — JRebel 의 무료 대안
- **Quarkus / Micronaut** — 다른 framework

→ 본 가이드는 DevTools auto-restart 권장 (2초 cycle 충분).

---

## 14. 자주 막히는 함정

### 함정 1 — DevTools 가 운영 JAR 에 포함
`developmentOnly` 가 아니라 `implementation` 으로 실수 → 운영에서도 동작. **`developmentOnly` 강제**.

### 함정 2 — IntelliJ 의 "Build project automatically" 비활성
class 파일 새로 안 생김 → 재시작 안 됨. **Settings 체크**.

### 함정 3 — Kotlin 컴파일이 느림
Java 컴파일보다 느림. `kotlin.daemon.jvmargs` 메모리 + parallel build + 작은 모듈 분리.

### 함정 4 — `application-local.yml` 의 환경변수 안 보임
Spring Boot 가 `local` profile 인지 모름. **`SPRING_PROFILES_ACTIVE=local`** export 또는 IntelliJ Run Configuration.

### 함정 5 — LiveReload + WebFlux 호환성
WebFlux (reactive) 환경에서 LiveReload 일부 깨짐. servlet 환경 기준.

### 함정 6 — DevTools restart 가 너무 잦음 (저장마다)
`quiet-period: 2s` 또는 IntelliJ 의 `Auto-save` interval 늘림.

### 함정 7 — `H2 console` (DevTools 기본) 가 보안 위험
운영에 절대 활성 X. 본 가이드는 PG 사용 — H2 console 무관.

### 함정 8 — Remote debug port 외부 노출
`5005` 가 외부에서 접근 가능 = JVM 제어. **localhost only** 또는 SSH tunnel.

### 함정 9 — Gradle daemon stale → build 실패
`./gradlew --stop` 후 재시도. 또는 IntelliJ `Invalidate Caches`.

### 함정 10 — `application.yml` 변경이 restart 안 트리거
`spring.devtools.restart.additional-paths` 에 추가 또는 default `src/main/resources` 안에 있는지 확인.

---

## 15. 다음 학습 로드맵 — 가이드 끝

✅ 10 단계 입문 + 기본 세팅 완주!

```
01. 설치 + Hello World        ← JDK / IDE / 첫 endpoint
02. 프로젝트 구조 + 설정       ← 패키지 / yaml / Config
03. Domain + JPA + Flyway     ← Entity / Repository / 마이그레이션
04. Controller + Validation   ← REST / DTO / 예외
05. Security + 배포           ← JWT / Argon2 / Docker
06. Swagger / OpenAPI         ← API 문서 자동
07. Controller 패턴           ← request/response 다양한 형식
08. Actuator + Monitoring     ← health / Prometheus
09. Logging + MDC             ← Logback / JSON log / request 추적
10. DevTools + Hot reload     ← 개발 cycle 가속  ← 지금
```

다음 단계:

| 단계 | 학습 | 노트 |
| --- | --- | --- |
| 11 | 응답 envelope 표준 깊이 | [[../common/response-envelope]] |
| 12 | Security 표준 깊이 | [[../common/security-config]] |
| 13 | 실전 레시피 (rate limit / file upload / webhook / 2FA / RBAC 등) | [[../api-design/api-design]] |
| 14 | JPA 깊이 (N+1 / 트랜잭션 / OSIV) | [[../database/jpa]] |
| 15 | MyBatis 같이 | [[../database/mybatis]] · [[../database/jpa-mybatis-coexist]] |
| 16 | Kotlin Coroutines + WebFlux (reactive) | (vault 외부 — 추후) |
| 17 | Spring Cloud (config / discovery / gateway) | (vault 외부 — MSA 도입 시) |

---

## 16. 참고

- [[guidelines|↑ Guidelines hub]]
- [[../kotlin-spring|↑ Kotlin Spring area]]
- Spring DevTools 공식: <https://docs.spring.io/spring-boot/reference/using/devtools.html>
- IntelliJ Spring Boot 가이드: <https://www.jetbrains.com/help/idea/spring-boot.html>
- Gradle 성능 가이드: <https://docs.gradle.org/current/userguide/performance.html>
