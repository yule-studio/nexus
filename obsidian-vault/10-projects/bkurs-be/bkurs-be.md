---
title: "bkurs-be — B-KURS 백엔드 프로젝트 hub"
kind: project
project: bkurs-be
agent: engineering-agent/tech-lead
status: in-progress
created_at: 2026-05-20T00:10:00+09:00
updated_at: 2026-05-22T00:40:00+09:00
tags: [project, backend, java-spring, mysql, bkurs]
home_hub: projects
related:
  - "[[../projects]]"
  - "[[../bkurs-fe/bkurs-fe]]"
  - "[[bkurs-be-log]]"
  - "[[handover/index]]"
  - "[[../../20-areas/backend/spring/java-spring/java-spring]]"
  - "[[../../20-areas/backend/spring/java-spring/database/database]]"
  - "[[../../20-areas/backend/spring/java-spring/api-design/api-design]]"
  - "[[../../20-areas/database/database]]"
---

# bkurs-be — B-KURS 백엔드 프로젝트 hub

> 본 hub 는 **프로젝트 정의 + 기술 결정 + 운영 원칙** 만. 코드 인수인계는 **[[handover/index]]**,
> PR / 사이클 history 는 **[[bkurs-be-log]]** 로 분리.

**[[../projects|↑ 10-projects]]** · **짝 프로젝트 [[../bkurs-fe/bkurs-fe]]**

## 진입

| 목적 | 문서 |
| --- | --- |
| 코드 인수인계 (전체 인덱스) | [[handover/index]] |
| 29 도메인 별 코드 설명 | [[handover/domains/auth\|domains/]] (29 doc) |
| PR / 사이클 진행 로그 | [[bkurs-be-log]] |
| 처음 합류자 — 역할별 진입 | [[handover/getting-started]] |
| 인증 / Security | [[handover/auth-and-security]] |
| 외부 SaaS plug-in | [[handover/external-gateways]] |
| 새 PR 시작 | [[handover/stacked-pr]] |
| Flyway 마이그레이션 정책 (SSoT) | [[policy/flyway-migration-policy]] |
| CI / 운영 troubleshooting | [[troubleshooting/ci-flyway-migration-policy-pr]] (PR #156 actuator/health) |

---

## 1. 프로젝트 정의

bkurs-fe (mock 으로 화면 완성된 React 프론트) 에 대응하는 **REST API 백엔드**.

inroad-high (원본) 는 Supabase 를 BaaS 로 직접 호출했지만, bkurs 는 별도 백엔드 서비스를 둠. 이유:

- 학원·학생·강사 도메인 정책을 코드로 명문화 (Supabase RLS 만으로 부족)
- 외부 SDK (Claude AI / STT / 카카오 SDK 등) 호출을 백엔드에서 통제
- 결제 / 영수증 같은 회계 데이터는 백엔드가 신뢰 경계

## 2. 기술 결정

| 영역 | 선택 | 이유 |
| --- | --- | --- |
| 언어 | Java 21 (LTS, Temurin) | 사용자 지정, 팀 친숙도 + record / pattern matching |
| 프레임워크 | Spring Boot 3.5.14 | 3.4 는 Initializr 거부, 3.5 가 최소 안정 |
| 빌드 | Gradle (Groovy DSL) + Wrapper | 팀 표준, IntelliJ 친화 |
| ORM | Spring Data JPA + Hibernate | 보일러플레이트 최소, repository 기반 빠른 개발 |
| 마이그레이션 | Flyway (+ flyway-mysql) | SQL 기반 + 버전 명확, 협업/롤백 용이 |
| DB | MySQL 9.6 (local) | 사용자 환경 기 설치 + 운영 표준 |
| 인증 | Spring Security + JWT (HS256) + Refresh rotation | 표준 토큰 / 세션 분기 가능 |
| 검증 | Bean Validation (Jakarta) | DTO 검증 단일화 |
| 로깅 | Logback + MDC requestId | 표준 + actuator 친화 + 분산 추적 |
| 보조 | Lombok, configuration-processor, actuator, springdoc-openapi | DTO/Entity 축소 + 헬스/메트릭 + Swagger |

상세 결정 근거는 [[handover/decisions/decision-java-21-spring-3-5-gradle-mysql]] + 도메인별 ADR
(`handover/decisions/*` 35개).

## 3. 로컬 환경

```bash
# Java 21
source ~/.sdkman/bin/sdkman-init.sh && sdk use java 21.0.5-tem

# MySQL local (또는 docker compose -f docker-compose.dev.yml up -d mysql)
brew services start mysql

# DB / user (1회만 — .env.local 의 DB_PASSWORD 와 동일하게)
mysql -uroot -p -e "
  CREATE DATABASE IF NOT EXISTS \`bkurs-local\` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
  CREATE USER IF NOT EXISTS 'mw'@'localhost' IDENTIFIED BY '\$DB_PASSWORD';
  GRANT ALL PRIVILEGES ON \`bkurs-local\`.* TO 'mw'@'localhost';
  FLUSH PRIVILEGES;
"

# env 주입 후 실행
cp .env.example .env.local && vi .env.local
export $(grep -v '^#' .env.local | xargs)
./gradlew bootRun        # profile=local
./gradlew clean check    # 빌드 + 통합 테스트
```

자세히 → [[handover/getting-started]], [[handover/env-deployment]].

자격 정보는 [[../../memory/project_bkurs_be_setup|project memory]] 와 .env.local (gitignore) 만
보관. **README / CI / 코드에 평문 비번 절대 금지.**

## 4. 작업 원칙

- **vault 기록**: [[../../memory/feedback_vault_project_tracking|정책]] 에 따라 PR 별 본 노트 + 20-areas 보강
- **commit 규칙**: 한국어 + gitmoji + 3 섹션 (변경 이유 / 주요 변경 / 비고) — [[../../memory/feedback_commit_format]]
- **PR 당 최소 3 commit**: docs-only chore 만 squash 허용 — [[../../memory/feedback_commit_splitting_policy]]
- **PR / Issue label + assignee 필수** — [[../../memory/feedback_pr_issue_labels_assignees]]
- **자동 머지 금지** — [[../../memory/feedback_no_auto_merge]]
- **author**: codwithyc 통일
- **민감 자격**: secret 절대 commit 금지 — `application-secret.yml` / `.env` 는 `.gitignore`

자세히 → [[handover/stacked-pr]].

## 5. 현재 상태 (요약)

| 항목 | 값 |
| --- | --- |
| dev 브랜치 | 사이클 2 완료 (commit 218a89b) — dev 배포 준비 완료 |
| 도메인 | 29 (functional 22 + gateway 5 + infra 2) |
| Endpoint | ~165 (cycle 1) → ~180+ (cycle 2 이후) |
| 회귀 | ~295 GREEN (cycle 1 종료) + cycle 2 추가 |
| Migration | V1 ~ V29 |
| 외부 SaaS | 5 abstraction (file/notification/pdf/ai/stt) + payment, 실 plug-in: PortOne / SendGrid / Claude / Clova |

상세 진행 history → [[bkurs-be-log]].

## 6. cross-link

- [[handover/index]] — 인수인계 문서 전체 인덱스 (필독)
- [[bkurs-be-log]] — PR / 사이클 진행 로그
- [[../bkurs-fe/bkurs-fe]] — 짝 프론트엔드 hub
- [[../../20-areas/backend/spring/java-spring/java-spring|Java + Spring Hub]]
- [[../../20-areas/database/database|database hub]]
- [[../../memory/project_bkurs_be_setup|project memory]]
