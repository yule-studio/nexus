---
title: "Spring Initializr — Boot 버전 선택 + Gradle DSL 비교"
kind: research
project: bkurs-be
session_id: bkurs-be-setup
created_at: 2026-05-20T00:20:00+09:00
agent: engineering-agent/tech-lead
status: complete
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-java-21-spring-3-5-gradle-mysql]]"
tags: [research, backend, spring, gradle]
---

# 질문

1. Spring Initializr 의 `bootVersion` 으로 어떤 값을 넘겨야 starter.zip 이 200 응답하는가?
2. Gradle Groovy DSL vs Kotlin DSL — Java 프로젝트에서 어느 것이 합리적인가?

# 조사

## 1. Boot 버전

처음 `bootVersion=3.4.1` 로 요청 → 400:

```json
{"timestamp":"...","status":400,"error":"Bad Request",
 "message":"Invalid Spring Boot version '3.4.1', Spring Boot compatibility range is >=3.5.0"}
```

→ Initializr 의 `/metadata/client` 로 확인:

```
['4.1.0.BUILD-SNAPSHOT', '4.1.0.RC1',
 '4.0.7.BUILD-SNAPSHOT', '4.0.6.RELEASE',
 '3.5.15.BUILD-SNAPSHOT', '3.5.14.RELEASE']
```

→ 3.5.14 가 최신 RELEASE — **선택**.

> **pitfall**: Initializr 의 compatibility range 는 정기적으로 올라간다. 6개월마다 starter.zip 요청 시 metadata 먼저 확인하는 게 안전. 옛 docs 의 버전을 hardcode 하면 400.

## 2. Gradle DSL

| 항목 | Groovy DSL | Kotlin DSL |
|---|---|---|
| 진입 | 익숙 (대부분 튜토리얼) | 학습 필요 |
| IDE 지원 | 강함 (IntelliJ 기본) | 강함 (IntelliJ 기본) |
| 타입 안전 | 약함 | 강함 |
| compile 시간 | 빠름 | Kotlin 컴파일 오버헤드 |
| Java 프로젝트 적합도 | 자연스러움 | 약간 어색 (build 만 Kotlin) |
| 커뮤니티 예제 | 압도적으로 많음 | 늘어나는 중 |

**Java 백엔드** 의 경우 Groovy 가 합리적. Kotlin 프로젝트면 Kotlin DSL 선택 (build script 도 한 언어 통일).

# 결론

- bootVersion = **3.5.14**
- buildTool = **gradle-project** (Groovy DSL)

# 참고

- Spring Initializr — https://start.spring.io
- [[../decisions/decision-java-21-spring-3-5-gradle-mysql]]
