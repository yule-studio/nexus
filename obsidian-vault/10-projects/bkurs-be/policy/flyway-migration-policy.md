---
title: "bkurs-be — Flyway 마이그레이션 정책"
kind: policy
project: bkurs-be
status: stable
created_at: 2026-06-01T00:00:00+09:00
updated_at: 2026-06-01T00:00:00+09:00
tags: [bkurs-be, db, flyway, migration, policy]
home_hub: "[[../bkurs-be]]"
related:
  - "[[../bkurs-be]]"
  - "[[../handover/index]]"
  - "[[../bkurs-be-log]]"
---

# bkurs-be — Flyway 마이그레이션 정책

> 본 문서는 **bkurs-be** 의 DB 마이그레이션 운영 정책 SSoT.
> 레포 README / BRANCH_STRATEGY / CHANGELOG 는 본 문서를 cross-link 한다 — 내용 복제 X.

## 1. 목적

- DB 스키마 변경을 **버전 관리 + 재현 가능** 하게 보관.
- 환경 (local / dev / stg / prod) 간 스키마 일관성 보장.
- 운영 적용 후의 변경을 **append-only** 로 강제해 history 손실 차단.

## 2. 적용 범위 / 디렉토리

| 위치 | 자동 실행 | 용도 |
| --- | --- | --- |
| `src/main/resources/db/migration` | ✅ Flyway 자동 (`classpath:db/migration`) | 일반 schema / data 마이그레이션 |
| `ops/db/backfill` | ❌ 수동 | 대용량 backfill SQL / 스크립트 |
| `ops/db/online` | ❌ 수동 | 대용량 alter / 긴 index — pt-osc / gh-ost 등 별도 도구 |
| `ops/db/runbooks` | ❌ — | 운영 절차 / rollback / 검증 가이드 (md) |

`src/main/resources/db/migration` 은 **단일 디렉토리 유지** — 도메인별 하위 디렉토리 금지.

### 왜 도메인별 디렉토리를 만들지 않나

- Flyway 는 실행 순서를 **파일명 사전 순** 으로 결정 (`Vyyyymmdd_HHmm` 가 그 순서를 보장).
- 도메인별 하위 디렉토리를 만들면:
  - 각 디렉토리의 V 파일 사이에 실행 순서 충돌 발생 가능.
  - 도메인 간 의존성 (예: course → enrollment → attendance) 이 디렉토리 경계 안에 갇힘.
  - `locations` 설정이 도메인 수만큼 늘어남 → 누락 가능성 ↑.
- 단일 디렉토리 + timestamp 네이밍 → 한 눈에 전체 history 가 보이고, 도메인 prefix
  (`__create_course_...`) 는 description 안에 표현.

## 3. 현재 시점 전제 (배포 전)

- **현재 어떤 환경에서도 Flyway 가 실행된 적 없다.** `flyway_schema_history` 테이블이
  어떤 DB 에도 존재하지 않는다.
- 그래서 기존 `V1__...` ~ `V29__...` 파일을 `VyyyyMMdd_HHmm__...` 으로 일괄 rename 가능
  (실제로 본 cycle 에서 그 작업을 수행).
- **최초로 공유 DB (개발자 공유 local 포함) / dev / stg / prod 중 하나라도 Flyway 가
  적용된 순간** 부터 본 자유는 사라진다 — 그 시점 이후 기존 V 파일 수정 / rename 금지.

## 4. 네이밍

### versioned migration

```
VyyyyMMdd_HHmm__lower_snake_case_description.sql
```

- prefix 는 반드시 대문자 `V`.
- 날짜시간은 작성 / 적용 시점. 동일 cycle 안에서는 분 단위로 증가시켜 unique 보장.
- description 은 `lower_snake_case` 유지 (`create_account`, `alter_academy_add_info_fields`).

예시:
```
V20260601_0000__init_schema.sql
V20260601_0001__create_account.sql
V20260601_0030__create_passage_interview.sql
```

### repeatable migration

```
R__lower_snake_case_description.sql
```

- versioned 모두 실행된 **후** 매번 실행 (체크섬 변경 시).
- VIEW / 함수 / 시드 데이터 등 idempotent 한 것에만.
- **현재 미사용.** 필요해지는 시점에만 추가.

## 5. 불변성 원칙 (배포 후)

최초로 어떤 환경에 Flyway 가 적용되어 `flyway_schema_history` 가 만들어진 순간부터:

1. **기존 V 파일 본문 수정 금지.** 체크섬 불일치로 다른 환경에서 startup fail.
2. **기존 V 파일 rename 금지.** 동일 사유.
3. **기존 V 파일 삭제 금지.** Flyway 가 missing migration 으로 fail.
4. **수정이 필요하면 새 V 파일을 추가.** (e.g. `V20260615_1430__alter_account_add_phone.sql`)
5. **rollback 도 새 V 파일.** 기존 V 를 되돌리지 말고 reverse 마이그레이션을 새로 추가.

> Flyway 설정 `spring.flyway.validate-on-migrate: true` 가 강제장치 — 체크섬 / 파일 누락 시
> startup fail. 본 정책의 코드 SSoT 가 바로 이 설정.

## 6. 대용량 backfill / online DDL 처리 기준

다음 중 하나에 해당하면 `src/main/resources/db/migration` 에 두지 말고
`ops/db/{backfill,online}` 으로 분리한다.

- **backfill**: 100k row 이상의 `UPDATE` / `INSERT` (운영 트래픽 차단 위험).
- **online DDL**: 대용량 테이블의 `ALTER` — MySQL metadata lock 으로 라이브 트래픽이
  몇 분 이상 막힐 가능성이 있을 때.
- **긴 index 생성**: `CREATE INDEX` 가 분 단위 이상 걸릴 때.
- **트랜잭션 분리 필요**: Flyway 는 기본 single transaction — 중간 실패 시 전체 롤백.
  partial 진행이 필요한 작업은 단일 트랜잭션 모델과 부적합.

이 기준에 해당하는 작업은:

1. `ops/db/online` 또는 `ops/db/backfill` 에 SQL / 스크립트 둠.
2. `ops/db/runbooks` 에 실행 절차 + rollback + 검증 SQL 정리.
3. 운영 적용 후, **빈 본문 또는 idempotent (`IF NOT EXISTS`) Flyway 마이그레이션** 을
   `src/main/resources/db/migration` 에 등록 — 다른 환경의 Flyway 가 스키마 차이를
   인식하도록.

## 7. PR 리뷰 체크리스트

마이그레이션이 포함된 PR 은 다음을 확인.

- [ ] 파일명이 `VyyyyMMdd_HHmm__...sql` 또는 `R__...sql`.
- [ ] description 이 `lower_snake_case`.
- [ ] **기존 V 파일을 수정 / rename / 삭제하지 않았다** (이미 어딘가에 적용된 경우).
- [ ] 단일 디렉토리 `src/main/resources/db/migration` 안에 있다 (하위 디렉토리 X).
- [ ] 새 entity / 변경 entity 와 `ddl-auto: validate` 가 startup 시 일치한다
      (JPA mapping mismatch 시 fail).
- [ ] 데이터를 채우는 `UPDATE` 가 대량 (100k+) 이면 `ops/db/backfill` 로 분리 검토.
- [ ] 대용량 테이블 alter / 긴 index 면 `ops/db/online` + runbook 분리 검토.
- [ ] 새 컬럼이 `NOT NULL` 인 경우 default 또는 backfill 전략이 명시되어 있다.
- [ ] FK 추가 시 lock 영향 검토 + 필요 시 ops/db/online 으로 분리.
- [ ] rollback 전략 — 새 V 파일로 reverse 가능한지 PR body 에 명시.
- [ ] 운영 적용 후라면, 본 PR 의 마이그레이션이 **새 V 파일** 인지 (기존 수정 X) 확인.

## 8. 배포 후 rollback / 수정 절차

운영 적용된 마이그레이션에 문제가 발견된 경우:

1. **기존 V 파일은 절대 수정하지 않는다.**
2. 새 V 파일을 추가해 reverse 또는 보정.
   - 예: `V20260615_1430__rollback_alter_account_phone.sql`
3. 데이터 보정이 필요하면 `ops/db/backfill` 에 SQL + `ops/db/runbooks` 에 절차.
4. 운영 적용 + 검증 후 CHANGELOG 에 기록.

## 9. 코드 SSoT

본 정책의 강제장치:

| 정책 | 코드 SSoT |
| --- | --- |
| 단일 디렉토리 자동 실행 | `application.yml` 의 `spring.flyway.locations: classpath:db/migration` |
| 체크섬 검증 / 누락 fail | `spring.flyway.validate-on-migrate: true` |
| JPA mapping vs schema 일치 | `spring.jpa.hibernate.ddl-auto: validate` |
| baseline 정책 | `spring.flyway.baseline-on-migrate: false` (빈 DB 가정. 기존 DB 위 첫 배포 시점에 재평가) |

위 4 설정은 `application-local.yml` / `application-dev.yml` / `application-prod.yml` 에서
유지되어야 한다. 변경 시 본 문서도 갱신.

## 10. 변경 이력

| 일자 | 변경 |
| --- | --- |
| 2026-06-01 | 정책 신설. V1~V29 timestamp rename 동시에. |
