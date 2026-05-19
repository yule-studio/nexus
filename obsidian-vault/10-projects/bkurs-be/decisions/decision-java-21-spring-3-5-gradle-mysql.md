---
title: "bkurs-be 기술 스택 결정 — Java 21 + Spring Boot 3.5 + Gradle + MySQL"
kind: decision
project: bkurs-be
session_id: bkurs-be-setup
created_at: 2026-05-20T00:20:00+09:00
agent: engineering-agent/tech-lead
status: decided
related:
  - "[[../bkurs-be]]"
  - "[[../research/research-spring-3-4-vs-3-5-gradle-dsl]]"
  - "[[../task-logs/task-log-setup-initial]]"
  - "[[../../../20-areas/backend/spring/java-spring/java-spring]]"
tags: [decision, backend, java-spring, mysql, gradle]
---

# 목표

bkurs-be 백엔드 시작 시 stack 단일 결정 — Java / Boot 버전 / 빌드 도구 / DB 종류.

# 결정

| 영역 | 결정 | 근거 |
|---|---|---|
| 언어 | **Java 21** (LTS, Temurin via SDKMAN: `21.0.5-tem`) | 사용자 지정. record / pattern matching / virtual thread 활용 가능 |
| 프레임워크 | **Spring Boot 3.5.14** | Spring Initializr 가 3.4.x 거부 (compatibility range `>=3.5.0`). 3.5 가 현재 가장 최신 LTS-grade |
| 빌드 | **Gradle (Groovy DSL) + Wrapper** | 팀 친숙도, IntelliJ 친화, Kotlin DSL 보다 진입 빠름 |
| DB | **MySQL 9.6** (local) | 사용자 환경 기 설치 (Homebrew). PostgreSQL 미설치 |
| ORM | Spring Data JPA + Hibernate | Spring 표준 + repository pattern 빠른 개발 |
| 마이그레이션 | Flyway + flyway-mysql | SQL 파일 명확, 협업 / 롤백 용이 |
| 인증 | Spring Security | 표준, 추후 JWT/세션 분기 가능 |
| 검증 | Bean Validation (Jakarta) | DTO 검증 단일화 |
| 보조 | Lombok / actuator / configuration-processor | 보일러플레이트 축소 + health/metric |

# 대안

- **Kotlin Spring** — 더 간결하지만 팀 표준 Java. 추가 학습 곡선이 작업 속도 저하. → **기각**
- **WebFlux (reactive)** — 학원 도메인은 cron-level 트래픽으로 reactive 의 이점 약함. blocking JPA 와도 충돌. → **기각**
- **MariaDB / PostgreSQL** — MariaDB 는 MySQL 호환이라 굳이. PostgreSQL 은 환경 추가 설치 필요. MySQL 9.6 이미 설치 → **선택**
- **Maven** — pom.xml 보일러플레이트 큼. → **기각**
- **Kotlin DSL Gradle** — Java 프로젝트엔 과함. → **기각**

# Why (4 구조)

**왜 필요한가**: 빈 repo 에 첫 commit 전에 stack 고정해야 폴더 구조 / 의존성 / CI 가 그에 맞게 잡힘. 도중 변경 시 모든 모듈 재구성 비용.

**안 하면 무엇이 깨지는가**: 후반 PR (B5~) 에서 stack 충돌 발견 시 — Boot 버전 mismatch → Hibernate 6 호환 깨짐 → 전체 entity 재작성. 비용 매우 큼.

**대안 비교**: 위 표 — 핵심은 사용자 환경 (Java SDK / MySQL) 과 팀 표준 (Java) 에 맞추기. 모든 대안은 추가 설치 / 학습 비용.

**트레이드오프**: Spring Boot 3.5 는 최신이라 일부 third-party (예: 옛 Querydsl) 호환 검증 필요. Boot 3.5 → Hibernate 6.6 → Querydsl 5.1.x 의존성 추가 시 검증 필수.

# 산출물

- `build.gradle` — 9 starter 의존성 (Spring Initializr 결과)
- `gradle-wrapper.properties` — Gradle 8.x
- `application.yml` — local profile + MySQL 연결 (`bkurs-local` DB, `mw` user)

# 참고

- [[../research/research-spring-3-4-vs-3-5-gradle-dsl]]
- [[../task-logs/task-log-setup-initial]]
- [[../../../20-areas/backend/spring/java-spring/java-spring]]
