---
title: "Flyway — SQL 기반 DB 마이그레이션 (Kotlin)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T14:30:00+09:00
tags: [knowledge, kotlin-spring, jpa, flyway, migration, database]
home_hub: database
related:
  - "[[database]]"
  - "[[jpa]]"
  - "[[../kotlin-spring]]"
  - "[[../../java-spring/database/flyway]]"
---

# Flyway — SQL 기반 DB 마이그레이션 (Kotlin)

**[[database|↑ DB hub]]** · **[[../../java-spring/database/flyway|↗ Java 원본]]**

> Flyway 자체는 SQL 기반이라 언어 무관. 본 노트는 Kotlin 프로젝트의
> `build.gradle.kts` 표기와 Kotlin 한정 함정만 정리. 본문 1, 3, 4, 5는
> Java 원본과 동일.

## 1. 왜 쓰는가

(Java 원본 [[../../java-spring/database/flyway#1 왜 쓰는가|§1]] 와 동일)

## 2. 설정

### Kotlin Gradle DSL 의존성

```kotlin
// build.gradle.kts
implementation("org.flywaydb:flyway-core")
implementation("org.flywaydb:flyway-mysql")          // MySQL 8+ 면 필수
implementation("org.flywaydb:flyway-database-postgresql")  // PG 15+ 면 필수
```

> **pitfall**: `flyway-mysql` 안 넣으면 MySQL 9 에서 `Unsupported Database: MySQL 9.x` 에러. `flyway-database-postgresql` 도 PG 15+ 에서 동일 함정.

### application.yml

(Java 원본 §2 와 동일 — yaml 은 언어 무관)

## 3. 파일 명명 규칙 / 4. 운영 패턴 / 5. 흔한 pitfall

(Java 원본 §3, §4, §5 와 동일)

## 6. Kotlin 한정 추가 함정

### 함정 6 — Flyway callback 의 Kotlin lambda
Java `FlywayConfigurationCustomizer customizer -> { ... }` 를 Kotlin 으로 옮길 때 `FlywayConfigurationCustomizer { it.callbacks(...) }` SAM conversion. `it` 위주.

### 함정 7 — `ddl-auto: validate` + Kotlin nullable 차이
Java entity 의 `String name` (non-null) 과 Kotlin entity 의 `var name: String?` 가 DB schema 의 `NOT NULL` 와 어긋날 수 있음. `var name: String = ""` default 또는 lateinit 권장.

### 함정 8 — H2 in-memory 테스트 의 dialect 차이
Kotlin `@DataJpaTest` 에서 H2 dialect 가 PG 와 미세하게 다름 — Flyway migration 의 PG-specific SQL (예: `GEOGRAPHY`, `JSONB`) 가 H2 에서 fail. **Testcontainers + 실제 PG** 권장.

### 함정 9 — Flyway 의 location 이 build/resources 와 src 분리
`src/main/resources/db/migration/V*.sql` 가 IDE 에서는 보이지만 Gradle build 시 누락 가능. `kotlin("plugin.spring")` 적용 모듈에서 `processResources` task 확인.

## 7. cross-link

- [[database]] / [[jpa]]
- [[../../java-spring/database/flyway|↗ Java 원본]]
- [[../kotlin-spring]]
- [[../api-design/file-upload-s3|file upload S3]]
