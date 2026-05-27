---
title: "01. 설치 + Spring Initializr + Hello World"
kind: guide
status: current
created_at: 2026-05-27T10:10:00+09:00
agent: engineering-agent/tech-lead
tags: [backend, kotlin-spring, guidelines, installation, hello-world]
---

# 01. 설치 + Spring Initializr + Hello World

**[[guidelines|↑ Guidelines hub]]** → 다음 [[02-project-structure]]

> JDK / Kotlin / IDE 설치 → Spring Initializr 로 첫 프로젝트 생성 → run.

---

## 1. JDK 21 설치

Spring Boot 3.x 는 **JDK 17+** 필요. 안정적인 LTS = **JDK 21**.

### 1.1 macOS

```bash
# Homebrew
brew install openjdk@21
brew install --cask temurin@21       # 또는 Eclipse Temurin (권장)

# PATH 추가 (~/.zshrc)
echo 'export JAVA_HOME=/Library/Java/JavaVirtualMachines/temurin-21.jdk/Contents/Home' >> ~/.zshrc
echo 'export PATH=$JAVA_HOME/bin:$PATH' >> ~/.zshrc
source ~/.zshrc

java -version
# openjdk version "21.0.x" 2024-xx-xx LTS
```

### 1.2 Ubuntu / Debian

```bash
sudo apt update
sudo apt install -y openjdk-21-jdk
java -version
```

### 1.3 Windows

[Temurin 21](https://adoptium.net/temurin/releases/) installer 다운로드 → 다음다음. PATH 자동.

### 1.4 SDKMAN! — Java 버전 관리 (권장)

여러 JDK 버전을 자유롭게 전환하고 싶다면:

```bash
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"

sdk list java                                # 사용 가능한 JDK 목록
sdk install java 21.0.4-tem                  # Temurin 21
sdk default java 21.0.4-tem

java -version
```

> 본 가이드는 **Temurin 21** 기준.

---

## 2. Kotlin

JDK 안에 Kotlin 컴파일러가 자동 포함되지 않음. 하지만 **Gradle / IDE 가 Kotlin 컴파일러를 자동 다운로드** 하므로 별도 설치 거의 불필요.

CLI 로 Kotlin script 돌리려면:

```bash
brew install kotlin            # macOS
sudo apt install -y kotlin     # Ubuntu
sdk install kotlin             # SDKMAN

kotlin -version                # Kotlin version 1.9.x
```

> **Spring Boot 프로젝트는 Kotlin CLI 불필요** — Gradle 의 `kotlin("jvm")` plugin 이 처리.

---

## 3. IDE — IntelliJ IDEA

**IntelliJ IDEA Ultimate** 가 Spring 진영 표준. Community Edition 도 가능하지만 Spring 통합 기능이 약함.

### 3.1 IntelliJ Community (무료) — 학습용

[다운로드](https://www.jetbrains.com/idea/download/) → install.

Spring Boot 프로젝트 import 하면 자동 인식.

### 3.2 IntelliJ Ultimate (유료) — 실무

학생 / 오픈소스는 [무료 라이선스](https://www.jetbrains.com/community/education/). 회사라면 회사 구매.

기능 차이:
- Spring Boot run dashboard
- Spring profile 인지
- HTTP client (Postman 대체)
- DB tool
- Profiler / Code coverage

### 3.3 VS Code (대안)

Java / Kotlin extension 설치 시 가능. 단 Spring Boot 작업엔 IntelliJ 우위.

---

## 4. Spring Initializr — 첫 프로젝트 생성

**Spring Initializr** = 공식 프로젝트 생성기. [start.spring.io](https://start.spring.io/) 에서 설정 후 다운로드.

### 4.1 웹에서 생성

[start.spring.io](https://start.spring.io/) 접속 후:

| 설정 | 값 |
| --- | --- |
| Project | **Gradle - Kotlin** |
| Language | **Kotlin** |
| Spring Boot | 3.3.x (최신 stable) |
| Project Metadata Group | `com.example` |
| Project Metadata Artifact | `memo-api` |
| Name | `memo-api` |
| Package name | `com.example.memo` |
| Packaging | **Jar** |
| Java | **21** |
| Dependencies | (다음 §4.2) |

### 4.2 Dependencies — 메모 API 에 필요한 것

`Add Dependencies` 클릭 후 추가:

- **Spring Web** — REST API
- **Spring Data JPA** — DB ORM
- **Spring Security** — 인증
- **Validation** — Bean Validation
- **PostgreSQL Driver** — PG 연결
- **Flyway Migration** — 마이그레이션
- **Spring Boot DevTools** — auto-reload (개발용)
- **Spring Boot Actuator** — healthcheck / metrics

`Generate` 버튼 → ZIP 다운로드 → 압축 풀기 → 디렉터리 진입.

### 4.3 CLI 로 생성 (대안 — 더 빠름)

```bash
# Spring Initializr CLI 직접 호출 (curl)
curl https://start.spring.io/starter.tgz \
  -d type=gradle-project-kotlin \
  -d language=kotlin \
  -d bootVersion=3.3.0 \
  -d baseDir=memo-api \
  -d groupId=com.example \
  -d artifactId=memo-api \
  -d name=memo-api \
  -d packageName=com.example.memo \
  -d packaging=jar \
  -d javaVersion=21 \
  -d dependencies=web,data-jpa,security,validation,postgresql,flyway,devtools,actuator \
  -o memo-api.tgz

tar -xvf memo-api.tgz
cd memo-api
```

### 4.4 생성된 구조

```
memo-api/
├── build.gradle.kts            # Gradle Kotlin DSL — 의존성 / plugin
├── settings.gradle.kts
├── gradlew                     # Gradle wrapper (Gradle 별도 설치 X)
├── gradlew.bat
├── gradle/
└── src/
    ├── main/
    │   ├── kotlin/
    │   │   └── com/example/memo/
    │   │       └── MemoApiApplication.kt
    │   └── resources/
    │       ├── application.properties     # 또는 application.yml
    │       └── static/ templates/
    └── test/
        └── kotlin/
            └── com/example/memo/
                └── MemoApiApplicationTests.kt
```

### 4.5 build.gradle.kts 핵심 부분

```kotlin
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

plugins {
    id("org.springframework.boot") version "3.3.0"
    id("io.spring.dependency-management") version "1.1.5"

    // ★ Kotlin 핵심 plugin 3개
    kotlin("jvm") version "1.9.24"
    kotlin("plugin.spring") version "1.9.24"            // @Component 자동 open
    kotlin("plugin.jpa") version "1.9.24"               // @Entity no-arg 생성자
}

group = "com.example"
version = "0.0.1-SNAPSHOT"

java {
    toolchain.languageVersion = JavaLanguageVersion.of(21)
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springframework.boot:spring-boot-starter-actuator")

    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")  // Kotlin data class 직렬화
    implementation("org.jetbrains.kotlin:kotlin-reflect")                  // Spring 필수

    runtimeOnly("org.postgresql:postgresql")
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")

    developmentOnly("org.springframework.boot:spring-boot-devtools")

    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
}

tasks.withType<KotlinCompile> {
    kotlinOptions {
        freeCompilerArgs += "-Xjsr305=strict"           // Java nullability annotation 엄격
        jvmTarget = "21"
    }
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

> **Kotlin 핵심 plugin 3개**:
> - `kotlin("jvm")` — Kotlin → JVM 바이트코드 컴파일.
> - `kotlin("plugin.spring")` — `@Component` / `@Service` / `@Configuration` 클래스를 자동 `open` (CGLIB proxy 호환).
> - `kotlin("plugin.jpa")` — `@Entity` 클래스에 no-arg 생성자 자동 합성 (Hibernate reflection 호환).

---

## 5. 첫 실행 — Hello World

### 5.1 자동 생성된 main 클래스

```kotlin
// src/main/kotlin/com/example/memo/MemoApiApplication.kt
package com.example.memo

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication

@SpringBootApplication
fun main(args: Array<String>) {
    runApplication<MemoApiApplication>(*args)
}

class MemoApiApplication                                // 비어있는 클래스 — 그저 type marker
```

> **`@SpringBootApplication`** = `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` 합성 어노테이션.
> **`runApplication<...>(*args)`** — Kotlin reified type. Java 의 `SpringApplication.run(MemoApiApplication.class, args)` 보다 짧음.

### 5.2 PostgreSQL 임시 실행 (DB 없으면 startup fail)

지금은 DB 가 없어서 실행하면 fail. 임시로 **H2 in-memory** 추가하거나, **PostgreSQL Docker** 띄움:

```bash
# Docker 가 있으면
docker run -d --name memo-db \
  -p 5432:5432 \
  -e POSTGRES_USER=memo \
  -e POSTGRES_PASSWORD=memo \
  -e POSTGRES_DB=memo \
  postgres:16
```

`src/main/resources/application.yml` (또는 application.properties) 작성:

```yaml
# src/main/resources/application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/memo
    username: memo
    password: memo
  jpa:
    hibernate:
      ddl-auto: validate              # Flyway 가 schema 관리
    open-in-view: false               # ⚠️ 무조건 false
  flyway:
    enabled: true
    baseline-on-migrate: true
```

마이그레이션 파일 placeholder 하나 (안 만들면 Flyway 가 baseline 못 잡음):

```sql
-- src/main/resources/db/migration/V1__init.sql
SELECT 1;     -- placeholder. 실 schema 는 다음 노트에서.
```

### 5.3 첫 endpoint

```kotlin
// src/main/kotlin/com/example/memo/HelloController.kt
package com.example.memo

import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable
import org.springframework.web.bind.annotation.RestController

data class Greeting(val message: String)


@RestController
class HelloController {

    @GetMapping("/")
    fun index(): Greeting = Greeting("Hello, Spring Boot!")

    @GetMapping("/hello/{name}")
    fun hello(@PathVariable name: String): Greeting =
        Greeting("Hello, $name!")
}
```

> **Kotlin idiom**:
> - `data class Greeting(val message: String)` — DTO 한 줄. Java 의 getter/setter/equals/hashCode 자동.
> - `fun index(): Greeting = Greeting(...)` — expression body. body `{ return ... }` 보다 짧음.
> - `@RestController` 가 final 이어도 `kotlin("plugin.spring")` 으로 자동 open.

### 5.4 임시 Security 비활성 (login 없이 호출하려면)

Spring Security 의존성 추가했으므로 default 로 모든 endpoint 가 401. 임시로 모두 허용:

```kotlin
// src/main/kotlin/com/example/memo/SecurityConfig.kt
package com.example.memo

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.config.annotation.web.invoke         // ★ Kotlin DSL
import org.springframework.security.web.SecurityFilterChain

@Configuration
class SecurityConfig {

    @Bean
    fun filterChain(http: HttpSecurity): SecurityFilterChain {
        http {
            csrf { disable() }
            authorizeHttpRequests { authorize(anyRequest, permitAll) }
        }
        return http.build()
    }
}
```

> **`org.springframework.security.config.annotation.web.invoke`** — Kotlin DSL 의 핵심 import. 없으면 `http { ... }` 가 안 됨.

### 5.5 실행

```bash
./gradlew bootRun

# 또는 IntelliJ 의 ▶ run 버튼

# log:
# Started MemoApiApplication in 2.5 seconds (process running for 3.1)
```

다른 터미널:

```bash
curl http://localhost:8080/
# {"message":"Hello, Spring Boot!"}

curl http://localhost:8080/hello/yuchan
# {"message":"Hello, yuchan!"}
```

---

## 6. 핵심 개념 정리

### 6.1 `@RestController` vs `@Controller`

- `@RestController` = `@Controller` + `@ResponseBody`. method 반환값이 자동으로 JSON.
- `@Controller` = template view (Thymeleaf 등). 본 가이드 범위 밖.

### 6.2 method 어노테이션

| 어노테이션 | HTTP method |
| --- | --- |
| `@GetMapping("/x")` | GET |
| `@PostMapping("/x")` | POST |
| `@PutMapping("/x")` | PUT |
| `@PatchMapping("/x")` | PATCH |
| `@DeleteMapping("/x")` | DELETE |
| `@RequestMapping("/x", method = [...])` | 옛 형식 |

### 6.3 path / query / body

```kotlin
@GetMapping("/users/{id}")                              // path
fun getUser(@PathVariable id: Long): User = ...

@GetMapping("/users")                                    // query
fun searchUsers(@RequestParam(required = false) name: String?, @RequestParam(defaultValue = "20") limit: Int): List<User> = ...

@PostMapping("/users")                                   // body
fun createUser(@RequestBody @Valid req: CreateUserRequest): User = ...
```

### 6.4 Spring 의 component 종류 (DI)

| 어노테이션 | 의미 |
| --- | --- |
| `@Component` | 가장 일반적 — Spring bean |
| `@Service` | business logic (의미만 다름) |
| `@Repository` | data access (의미만 다름) |
| `@Controller` / `@RestController` | web layer |
| `@Configuration` | bean 설정 클래스 |

본 가이드는 `@Service` / `@Repository` / `@RestController` 위주.

---

## 7. 자주 막히는 함정 (초보)

### 함정 1 — `kotlin("plugin.spring")` 누락
final class 라 Spring CGLIB proxy 못 만듦 → `@Transactional` 동작 X, `@Service` lazy 못 됨. **build.gradle.kts 의 plugin 블록 확인**.

### 함정 2 — `org.springframework.security.config.annotation.web.invoke` import 누락
`http { csrf { } }` Kotlin DSL 이 안 됨 → 컴파일 에러. **import 추가**.

### 함정 3 — PostgreSQL 안 띄우고 실행
`Connection refused`. **`docker run postgres:16`** 먼저.

### 함정 4 — `ddl-auto: create` / `update` 사용
운영에서 위험. **항상 `validate` + Flyway**.

### 함정 5 — `ddl-auto: validate` + 마이그레이션 파일 없음
DB 가 비어있어 schema validation fail. **V1__init.sql placeholder** 라도 만들기.

### 함정 6 — IntelliJ 가 Kotlin code import 못 함
`./gradlew build --refresh-dependencies` → IntelliJ `Reload Gradle Project`.

### 함정 7 — `runApplication<MemoApiApplication>(*args)` 의 `*`
Kotlin spread operator — `args: Array<String>` 을 vararg 로 변환. 빠뜨리면 컴파일 에러.

### 함정 8 — Gradle daemon stale
build 실패 시 `./gradlew --stop` 후 재시도.

---

## 8. 다음

✅ 설치 + Hello World 끝.

→ [[02-project-structure]] — 본격 프로젝트 패키지 구조 + application.yml + Config 클래스
