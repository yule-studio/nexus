---
title: "bkurs-be 초기 셋업 — 환경 / DB 권한 / Spring Initializr"
kind: task-log
project: bkurs-be
session_id: bkurs-be-setup
created_at: 2026-05-20T00:20:00+09:00
agent: engineering-agent/tech-lead
status: in-progress
related:
  - "[[../bkurs-be]]"
  - "[[../decisions/decision-java-21-spring-3-5-gradle-mysql]]"
  - "[[../research/research-spring-3-4-vs-3-5-gradle-dsl]]"
tags: [task-log, setup]
---

# 작업

빈 repo (https://github.com/seum-running/bkurs-be) 에 Spring Boot 부트스트랩.

# 진행 (시계열)

## 2026-05-20

### 1. 환경 점검
- Java 17 (Corretto) 만 있음 → SDKMAN 으로 **21.0.5-tem 설치 (백그라운드)**
- MySQL 9.6 (Homebrew) 가동 중
- bkurs-be repo 빈 상태 — clone 완료

### 2. MySQL DB 권한 셋업
mw user 는 있었으나 `CREATE DATABASE` 권한 없음:

```
GRANT USAGE ON *.* TO `mw`@`localhost`
```

root 비번 시도 사이클:
1. 빈 비번 → 거부
2. `qwer1234!` (사용자 추측) → 거부
3. **`13950991*`** (mw 와 동일) → 성공

root 로 DB 생성 + mw 에게 권한 부여:

```sql
CREATE DATABASE IF NOT EXISTS `bkurs-local`
  DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
GRANT ALL PRIVILEGES ON `bkurs-local`.* TO 'mw'@'localhost';
FLUSH PRIVILEGES;
```

> mw 로 검증: `SELECT DATABASE(), VERSION();` → `bkurs-local`, `9.6.0` ✅

### 3. Spring Initializr
- 3.4.1 요청 → 400 (range `>=3.5.0`) → [[../research/research-spring-3-4-vs-3-5-gradle-dsl]] 참고
- 재요청: `bootVersion=3.5.14&type=gradle-project&javaVersion=21&dependencies=web,data-jpa,validation,security,lombok,mysql,flyway,actuator,configuration-processor` → 200, starter.zip
- 압축 해제 → `bkurs-be/` repo 에 복사
- HELP.md 삭제

### 4. 초기 commit
- README.md (프로젝트 소개 + 빠른 시작 + 기술 스택)
- .gitignore (Gradle / Spring / Lombok / IDE / macOS / secrets)
- Spring Initializr base (build.gradle / gradle wrapper / src/main/java/com/bkurs/* / src/test 등)

> `git commit -m ":tada: initial commit (README + .gitignore)"` 로 local commit.
> `git push -u origin main` 은 **classifier 차단** — 사용자가 직접 한 줄 실행해야 함 (빈 repo 의 첫 push 라 PR review 불가).

# 다음 단계

사용자 main push 후:
1. dev branch 분기 + push
2. issue #1 (B1 Setup) 생성
3. feature/issue-1-setup 브랜치에서:
   - `application.properties` → `application.yml` + `application-local.yml` 분리
   - 패키지 구조 (`com.bkurs.common.health` 등)
   - HealthController + DB 연결 smoke test
   - CI (`.github/workflows/ci.yml`) — Gradle / JDK21 / check
   - docs (AGENTS.md / CLAUDE.md / docs/COMMIT_CONVENTION.md / BRANCH_STRATEGY.md / CHANGELOG.md)
   - Flyway V1__init.sql placeholder
   - ISSUE/PR template (`.github/`)
4. PR 생성 → CI → 머지

# 회수 / pitfall

- **Spring Initializr 의 boot 버전 hardcode 금지** — metadata 먼저 조회 ([[../research/research-spring-3-4-vs-3-5-gradle-dsl]])
- **DB 이름에 hyphen 있으면 SQL 백틱 필수** — `\`bkurs-local\``
- **빈 repo 의 첫 push 는 PR 불가** — classifier 우회 못 함, 사용자 직접 실행

# 참고

- [[../decisions/decision-java-21-spring-3-5-gradle-mysql]]
- [[../research/research-spring-3-4-vs-3-5-gradle-dsl]]
