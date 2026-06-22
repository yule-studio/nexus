---
title: "DB 설계 실무"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
updated_at: 2026-06-22
tags:
  - area
  - database
  - database-design
---

# DB 설계 실무

> 정의: 도메인을 관계형 스키마로 옮기는 작업 — 정규화로 데이터 무결성을 확보하고, ERD로 구조를 표현하며, 인덱스 전략과 마이그레이션 원칙으로 운영 가능한 형태를 만든다.

**[[../database|↑ database hub]]**

## 1. 무엇인가

DB 설계는 비즈니스 도메인의 엔터티와 관계를 관계형 모델(테이블/컬럼/제약/관계)로 사상(mapping)하는 작업이다. 다루는 범위는 다음 네 축이다.

- **정규화 (Normalization)** — 중복과 갱신 이상을 제거하기 위한 스키마 분해. 1NF ~ 3NF, BCNF.
- **ERD (Entity-Relationship Diagram)** — 엔터티/속성/관계(1:1, 1:N, N:M)와 카디널리티를 표기하는 설계 언어.
- **인덱스 전략** — 조회 패턴에 맞춘 B-Tree/복합/부분/커버링 인덱스 선택과 PK/FK 제약.
- **마이그레이션** — 스키마를 버전 관리하고 무중단으로 진화시키는 변경 절차.

물리 모델(스토리지/파티셔닝)보다 논리 모델(무결성/관계)에 무게를 두며, 튜닝(쿼리 성능 개선)은 [[../database-tuning/database-tuning|DB 튜닝]] 노트가 담당한다.

## 2. 왜 / 언제

**(1) 왜 필요한가** — 스키마는 한 번 운영에 들어가면 바꾸기가 가장 어려운 자산이다. 코드는 배포로 롤백되지만 데이터는 누적되어 사라지지 않는다. 초기 설계가 무결성을 강제하지 않으면 잘못된 데이터가 쌓이고, 이후 어떤 애플리케이션 로직으로도 완전히 정정할 수 없다.

**(2) 안 하면 무슨 문제 — 구체 예시**
- **갱신 이상(update anomaly)**: 주문 테이블에 고객 이름/주소를 그대로 중복 저장하면, 고객이 이사할 때 그 고객의 모든 주문 행을 갱신해야 한다. 일부만 갱신되면 같은 고객의 주소가 행마다 달라진다.
- **삽입 이상(insertion anomaly)**: 상품 정보를 주문 테이블에만 두면, 아직 주문이 없는 신규 상품을 등록할 곳이 없다.
- **느린 조회**: 인덱스 없는 `WHERE email = ?` 는 수백만 행 풀스캔(Seq Scan)을 유발해 로그인 응답이 초 단위로 느려진다.
- **참조 깨짐**: FK 제약이 없으면 `orders.user_id` 가 삭제된 사용자를 가리키는 고아 행(orphan row)이 생긴다.

**(3) 대안** — NoSQL(문서/키값)로 비정규화 모델을 쓰거나, 읽기 전용 집계는 별도 OLAP 스토어/머티리얼라이즈드 뷰로 분리하는 방법이 있다. 다만 관계 무결성과 트랜잭션이 중요한 핵심 도메인은 RDBMS 정규화 설계가 여전히 기본값이다.

**(4) 트레이드오프 — 정규화 vs 비정규화**
- 정규화는 중복을 제거해 무결성과 쓰기 효율을 높이지만, 조회 시 JOIN이 늘어 읽기 비용이 증가한다.
- 비정규화는 읽기를 빠르게 하지만 중복 데이터의 일관성을 애플리케이션/트리거가 책임져야 한다.
- 실무 기준: **3NF로 설계한 뒤, 측정된 병목에 한해 선택적으로 비정규화**한다. 추측으로 미리 비정규화하지 않는다.

## 3. 핵심 개념 / 어떻게

### 정규화 (1NF ~ BCNF)

- **1NF**: 모든 컬럼이 원자값(atomic). `phones = '010-1, 010-2'` 같은 다중값 컬럼을 금지하고 별도 행/테이블로 분리.
- **2NF**: 1NF + 부분 함수 종속 제거. 복합 PK `(order_id, product_id)` 테이블에서 `product_name` 이 `product_id` 에만 종속되면 분리.
- **3NF**: 2NF + 이행 종속 제거. `users(id, zip, city)` 에서 `city` 가 `zip` 에 종속되면 `zips(zip, city)` 로 분리.
- **BCNF**: 모든 결정자가 후보키. 3NF에서 드물게 남는 결정자 이상을 제거하는 더 엄격한 형태.

예시 (3NF 위반 → 분리):

```sql
-- 위반: city 가 zip 에 이행 종속
CREATE TABLE users (id BIGINT PK, name TEXT, zip TEXT, city TEXT);
-- 개선
CREATE TABLE users (id BIGINT PRIMARY KEY, name TEXT, zip TEXT REFERENCES zips(zip));
CREATE TABLE zips  (zip TEXT PRIMARY KEY, city TEXT);
```

### ERD 표기

- 엔터티 = 사각형, 속성 = 컬럼, 관계 = 선. 카디널리티는 까마귀발(crow's foot)로 1:1 / 1:N / N:M 표현.
- N:M 은 RDBMS에 직접 표현할 수 없으므로 **연결 테이블(junction table)** 로 분해한다: `student_courses(student_id, course_id)` 복합 PK + 각각 FK.

### 인덱스 종류

- **B-Tree**: 기본값. 등호/범위/정렬(`=`, `<`, `BETWEEN`, `ORDER BY`)에 사용.
- **복합(composite)**: `(a, b)` 인덱스는 `WHERE a=?` 와 `WHERE a=? AND b=?` 에 효과적이나 `WHERE b=?` 단독에는 안 탄다(좌측 접두 규칙, leftmost prefix).
- **부분(partial)**: 조건부 인덱스로 인덱스 크기 절감. 예: `CREATE INDEX ON orders(user_id) WHERE status = 'open';`
- **커버링(covering)**: 인덱스만으로 쿼리를 해결(테이블 접근 생략). PostgreSQL `INCLUDE` 또는 MySQL의 인덱스 내 컬럼 활용.

```sql
-- 커버링: status 로 필터하고 created_at 정렬 결과를 인덱스만으로 반환
CREATE INDEX idx_orders_status_created
  ON orders (status, created_at) INCLUDE (total);
```

### 마이그레이션 원칙

- **버전 관리**: Flyway/Liquibase/Alembic 등으로 모든 스키마 변경을 코드로 추적. 수동 ALTER 금지.
- **전진 전용 + 안전한 단계**: 컬럼 추가는 NULL 허용 또는 DEFAULT 로 시작, 백필 후 NOT NULL 승격. 컬럼 삭제는 코드에서 사용 중지 → 배포 안정화 → 삭제의 2단계 배포.
- **무중단**: 대형 테이블의 인덱스는 PostgreSQL `CREATE INDEX CONCURRENTLY`, MySQL `ALGORITHM=INPLACE` 로 락 최소화.
- **롤백 가능성**: 파괴적 변경(DROP)은 별도 마이그레이션으로 분리해 되돌릴 여지를 남긴다.

## 4. 적용 / 함정

- **FK 인덱스 누락**: 대부분의 RDBMS는 FK 컬럼에 인덱스를 자동 생성하지 않는다(MySQL InnoDB는 예외적으로 생성). 부모 행 삭제/JOIN 시 풀스캔이 발생하므로 직접 인덱스를 건다.
- **과도한 인덱스**: 인덱스마다 INSERT/UPDATE 비용과 디스크가 늘어난다. 안 쓰는 인덱스는 `pg_stat_user_indexes.idx_scan = 0` 으로 찾아 제거.
- **자연키를 PK로**: 이메일/주민번호 같은 자연키는 변경 가능하고 노출 위험이 있다. 대리키(surrogate key, BIGINT/UUID)를 PK로 두고 자연키에는 UNIQUE 제약을 건다.
- **NULL 의미 오용**: "값 없음"과 "0"을 NULL로 뭉개면 집계가 틀어진다. NOT NULL + DEFAULT 를 기본으로 검토.
- **타임존 없는 시각**: `TIMESTAMP` 대신 `TIMESTAMPTZ`(PostgreSQL)를 사용해 UTC 저장을 강제.
- **N:M 을 콤마 문자열로**: 1NF 위반. 반드시 연결 테이블로.

## 5. 관련

- [[../database-tuning/database-tuning|DB 튜닝]] — 설계한 스키마의 쿼리/인덱스 성능을 측정·개선
- [[../postgresql/postgresql|PostgreSQL]] / [[../mysql/mysql|MySQL]] — 엔진별 구체 구현
- [[../../../30-resources/computer-science/database-theory/database-theory|database-theory]] — 관계 대수/함수 종속 이론 배경
- [[../database|↑ database hub]]
- 외부 참고: PostgreSQL 공식 문서(Indexes, DDL), Use The Index, Luke (use-the-index-luke.com), C.J. Date "An Introduction to Database Systems"
