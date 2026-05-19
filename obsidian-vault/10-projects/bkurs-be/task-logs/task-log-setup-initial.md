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

### 5. dev / feature 분기 + setup PR 작업 (local)

main push 대기 사이에 local 로 진행 — push 시 한 번에 GitHub 반영.

```
* feature/issue-1-setup (5 commit)
  ├─ e39c560 :whale: docker-compose.yml — MySQL 컨테이너 한 줄 가동 + README 옵션 A/B
  ├─ 777eb2e :memo: docs / CI / PR 템플릿 / editorconfig
  ├─ 39a2b09 :sparkles: HealthController + SecurityConfig + Flyway V1 + smoke test
  ├─ 4138a18 :wrench: application.yml 분리 (공통 + local profile) + DB 연결 설정
  └─ 53c50f7 :tada: initial commit — README + .gitignore + Spring Boot 3.5 base
dev (53c50f7 와 동일)
main (53c50f7 와 동일)
```

각 commit 의 변경:

| commit | 핵심 변경 |
|---|---|
| `:wrench:` application.yml | application.properties 삭제 + yml 2개 (공통 / local) + HikariCP + JPA validate + Flyway 활성화 + actuator 노출 |
| `:sparkles:` Health + Security + Flyway + tests | HealthController GET /api/v1/health + SecurityConfig stateless permitAll + V1__init_schema.sql placeholder + 4 smoke 테스트 (context / dataSource / health / actuator) |
| `:memo:` docs | AGENTS / CLAUDE / COMMIT / BRANCH / CHANGELOG / ci.yml (MySQL 9 service) / PR template / ISSUE template / .editorconfig |
| `:whale:` docker-compose | mysql:9.0 + bkurs-local + mw user 자동 셋업 + healthcheck + named volume. README 의 빠른 시작 = 옵션 A (Docker) / B (로컬 설치) 분리 |

### 6. 테스트 통과 (local)

```
$ ./gradlew test --no-daemon
> Task :test
BUILD SUCCESSFUL in 14s
```

4 테스트 (contextLoads / dataSourceCanConnect / healthEndpointReturnsOk / actuatorHealthIsUp) 모두 GREEN.

# 다음 단계

사용자 한 줄 push 후 자동 진행:
```
cd ~/masterway-dev/bkurs-be && \
  git push -u origin main && \
  git push -u origin dev && \
  git push -u origin feature/issue-1-setup
```

자동 후속:
1. `gh issue create` — B1 Setup
2. `gh pr create --base dev --label enhancement --assignee codwithyc`
3. CI (MySQL 9 service + JDK21 + ./gradlew clean check) 통과 대기
4. `gh pr merge --merge --delete-branch`
5. `gh issue close`
6. 본 task-log 진행 로그에 PR 머지 결과 추가
7. [[../bkurs-be|hub]] 의 "진행 로그" 표 업데이트

# 회수 / pitfall (누적)

- **Spring Initializr 의 boot 버전 hardcode 금지** — metadata 먼저 조회. vault [[../../../20-areas/backend/spring/java-spring/pitfalls/spring-initializr-version]]
- **DB 이름에 hyphen 있으면 SQL 백틱 필수** — `\`bkurs-local\``
- **빈 repo 의 첫 push 는 PR 불가** — classifier 우회 못 함, 사용자 직접 실행
- **Spring Boot 3.5 + MySQL 9 = flyway-mysql 필수** — flyway-core 만으로는 `Unsupported Database` 에러. vault [[../../../20-areas/backend/spring/java-spring/database/flyway]]
- **ddl-auto: validate 가 정답** — create / update 는 운영 schema drift. Flyway 와 충돌

# 참고

- [[../decisions/decision-java-21-spring-3-5-gradle-mysql]]
- [[../research/research-spring-3-4-vs-3-5-gradle-dsl]]
