---
title: "Flyway — SQL 기반 DB 마이그레이션"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-20T00:30:00+09:00
tags: [knowledge, spring, jpa, flyway, migration, database]
home_hub: database
related:
  - "[[database]]"
  - "[[jpa]]"
  - "[[../java-spring]]"
  - "[[../../../../../10-projects/bkurs-be/bkurs-be]]"
---

# Flyway — SQL 기반 DB 마이그레이션

**[[database|↑ DB hub]]**

## 1. 왜 쓰는가

JPA 의 `spring.jpa.hibernate.ddl-auto=create` 또는 `update` 는 다음 이유로 운영에 부적합:

- **무엇이 바뀌는지 commit 직전까지 모름** — 코드 diff 만 보고 DB 변경을 추측해야 함
- **다중 서버 / 다중 환경에서 동기화 보장 X** — 운영/스테이징 schema 가 미세하게 어긋남
- **롤백 불가** — `update` 는 DROP 안 함, 한 번 잘못 add 한 컬럼이 영구 남음
- **데이터 정합 보장 X** — 컬럼 추가 시 default / not-null 정책 코드로 표현 안 됨

Flyway 는 SQL 파일을 버전 순서로 적용 + `flyway_schema_history` 테이블에 기록 → 어느 환경이 어느 버전인지 SELECT 한 줄로 확인.

## 2. 설정

### Spring Boot 의존성

```gradle
implementation 'org.flywaydb:flyway-core'
implementation 'org.flywaydb:flyway-mysql'  // MySQL 8+ 면 필수
```

> **pitfall**: `flyway-mysql` 안 넣으면 MySQL 9 에서 `Unsupported Database: MySQL 9.x` 에러. PostgreSQL / H2 는 core 만으로 충분.

### application.yml

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration  # 기본값
    baseline-on-migrate: false         # 빈 DB 면 false (false 가 안전, true 면 기존 schema 보존)
    validate-on-migrate: true          # 변경된 V1 다시 적용 시 fail
  jpa:
    hibernate:
      ddl-auto: validate               # Flyway 와 같이 쓸 때 validate 만!
```

`ddl-auto: validate` 가 핵심 — entity 와 schema 가 어긋나면 startup fail (조용한 drift 방지).

## 3. 파일 명명 규칙

```
src/main/resources/db/migration/
├── V1__init_schema.sql
├── V2__add_student_table.sql
├── V3__index_email.sql
└── R__seed_default_codes.sql  (반복 실행 — 기준값 update)
```

- `V<N>__<설명>.sql` — 한 번만 실행 (예: V1__init.sql)
- `R__<설명>.sql` — checksum 변경 시 매번 재실행 (예: 코드 마스터 데이터)
- `<N>` 은 숫자 또는 `2026.05.20.1` 같은 timestamp — 팀 규칙

## 4. 운영 패턴

### V1 placeholder 로 시작

새 프로젝트는 첫 commit 에 `V1__init_schema.sql` 빈 파일 + 주석만:

```sql
-- V1__init_schema.sql
-- 빈 placeholder. 실 schema 는 첫 도메인 entity 도입 시 V2 부터.
SELECT 1;
```

이유: Flyway 가 처음 startup 시 baseline 을 잡음. V1 이 없으면 baseline-on-migrate 활성화해야 하는데 그건 위험.

### 마이그레이션 = 코드 review 대상

- DDL 도 PR review — `ALTER TABLE` 한 줄도 회의록 → SQL 변환 명확
- entity 변경과 SQL 변경이 한 commit 에 — 분리되면 reviewer 가 mismatch 못 잡음

### 롤백

Flyway 는 자동 down migration 없음. 롤백 시:
- 작은 변경: V<next>__revert.sql 로 down 작성
- 큰 변경: DB 복원 (백업에서)

## 5. 흔한 pitfall

| pitfall | 증상 | 해결 |
|---|---|---|
| flyway-mysql 누락 | `Unsupported Database: MySQL 9.x` | `implementation 'org.flywaydb:flyway-mysql'` 추가 |
| V1 checksum 변경 | `Migration checksum mismatch` | 옛 V1 절대 수정 금지 — 새 V<N> 추가 |
| ddl-auto: update | Flyway 적용 후에도 Hibernate 가 DDL 변경 | `ddl-auto: validate` 또는 `none` |
| 운영 DB 에 V1 가 잘못 들어감 | flyway_schema_history 에 실패 기록 | `flyway repair` 또는 row 삭제 후 재실행 |
| dev / prod 의 V1 가 다른 적용 시각 | schema drift | CI 에서 `flyway info` 자동 비교 |

## 6. cross-link

- [[database]] / [[jpa]]
- [[../java-spring]]
- [[../../../../../10-projects/bkurs-be/bkurs-be]] — 본 노트가 활용될 첫 프로젝트
- [[../api-design/file-upload-s3|file upload S3 — migration vs metadata 분리]]
