---
title: "DB 튜닝"
kind: knowledge
project: database
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T04:00:00+09:00
updated_at: 2026-06-22
tags:
  - area
  - database
  - database-tuning
---

# DB 튜닝

> 정의: 측정 가능한 병목을 기준으로 쿼리 실행 계획, 인덱스, 통계, 캐시 계층을 조정해 응답 시간과 처리량을 개선하는 작업.

**[[../database|↑ database hub]]**

## 1. 무엇인가

DB 튜닝은 "느리다"를 "어디가, 왜, 얼마나 느린지"로 바꾼 뒤 가장 비용 대비 효과가 큰 지점을 고치는 활동이다. 다루는 범위는 다음과 같다.

- **쿼리 최적화** — 실행 계획(EXPLAIN)을 읽고 풀스캔/잘못된 JOIN 순서/N+1을 제거.
- **인덱스 튜닝** — 인덱스를 추가/삭제/재설계하고, 인덱스가 안 타는 원인을 진단.
- **통계와 유지보수** — 옵티마이저가 의존하는 통계(ANALYZE)와 VACUUM/dead tuple 관리.
- **캐시 계층** — shared_buffers/buffer pool, OS 페이지 캐시, 애플리케이션 캐시.

설계 단계의 정규화/스키마 결정은 [[../database-design/database-design|DB 설계 실무]] 노트가 담당한다. 튜닝은 그 위에서 운영 중 발생하는 성능 문제를 다룬다.

## 2. 왜 / 언제

**(1) 왜 필요한가** — 데이터가 커지면 같은 쿼리도 선형 이상으로 느려진다. 100행에서 빠르던 풀스캔은 1,000만 행에서 타임아웃이 된다. 옵티마이저는 통계 기반 추정으로 동작하므로, 통계가 낡거나 인덱스가 없으면 잘못된 계획을 선택한다.

**(2) 안 하면 무슨 문제 — 구체 예시**
- **느린 쿼리**: `WHERE LOWER(email) = ?` 는 `email` 인덱스를 못 타고 Seq Scan + 매 행 함수 호출로 수 초가 걸린다.
- **N+1**: 게시글 목록 1회 + 각 글의 작성자를 행마다 1회씩 조회하면, 글 50개에 51번의 쿼리가 나가 네트워크 왕복이 응답을 지배한다.
- **락 경합**: 긴 트랜잭션이 자주 갱신되는 행을 잡고 있으면 다른 요청이 `lock wait` 로 줄을 선다. PostgreSQL은 dead tuple 누적 + autovacuum 지연으로 테이블 bloat 까지 겹친다.
- **계획 퇴화**: 통계가 낡아 옵티마이저가 1만 행을 1행으로 오판하면, 인덱스가 있어도 Nested Loop를 잘못 선택해 폭주한다.

**(3) 대안** — 읽기 부하는 read replica로 분산하거나, 무거운 집계는 머티리얼라이즈드 뷰/사전 집계 테이블로 옮기거나, 핫 데이터를 Redis 등 외부 캐시로 뺄 수 있다. 다만 이들 전에 쿼리/인덱스부터 고치는 것이 가장 저렴하다.

**(4) 트레이드오프**
- **인덱스 추가**: 읽기를 빠르게 하지만 INSERT/UPDATE/DELETE마다 인덱스 갱신 비용과 디스크가 늘어난다. 쓰기 많은 테이블에 인덱스를 남발하면 전체가 느려진다.
- **캐시**: 읽기 지연을 줄이지만 무효화(invalidation) 정합성 문제를 만든다.
- **비정규화/사전 집계**: 조회는 빨라지나 갱신 일관성 부담이 생긴다. 측정 후 도입한다.

## 3. 핵심 개념 / 어떻게

### EXPLAIN / EXPLAIN ANALYZE 읽는 법

- `EXPLAIN` 은 옵티마이저의 **추정 계획**만, `EXPLAIN ANALYZE` 는 **실제 실행**(시간/실제 행 수)을 보여준다. 진단은 ANALYZE로.
- 핵심 신호:
  - `Seq Scan` on 대형 테이블 + 선택적 `WHERE` → 인덱스 부재 의심.
  - **estimated rows vs actual rows** 큰 차이 → 통계가 낡음(ANALYZE 필요).
  - `Nested Loop` 의 안쪽이 대량 행 → JOIN 전략/인덱스 재검토.
  - 상위 노드 시간 - 하위 노드 시간 = 그 노드 자체 비용.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE user_id = 42 AND status = 'open';
-- Seq Scan on orders ... rows=... actual time=...
-- → idx (user_id, status) 추가 후 Index Scan 으로 전환되는지 확인
```

### 인덱스가 안 타는 케이스

- **컬럼에 함수/연산 적용**: `WHERE LOWER(email)=?`, `WHERE created_at::date = ?`. 해결: 표현식 인덱스(`CREATE INDEX ON t (LOWER(email))`) 또는 범위 조건으로 재작성.
- **선두 와일드카드 LIKE**: `LIKE '%term'` 은 B-Tree를 못 탄다. 전문 검색(GIN/trigram, `pg_trgm`)으로.
- **암묵적 타입 캐스팅**: `WHERE varchar_col = 123` 처럼 타입 불일치 시 인덱스 미사용.
- **복합 인덱스 좌측 접두 위반**: `(a, b)` 인덱스에 `WHERE b=?` 단독은 안 탄다.
- **낮은 선택도**: `status='active'` 가 전체의 95%면 옵티마이저가 일부러 Seq Scan을 고른다(정상 동작).

### N+1 제거

ORM에서 흔하다. 지연 로딩 대신 JOIN 또는 일괄 조회로 묶는다.

```sql
-- N+1: 글마다 작성자 별도 조회
-- 개선: 한 번에 JOIN
SELECT p.*, u.name FROM posts p JOIN users u ON u.id = p.user_id
WHERE p.board_id = ?;
```

ORM에서는 eager load(JPA `fetch join`, Django `select_related`, Rails `includes`)로 해결.

### 통계 / VACUUM

- **통계**: 옵티마이저는 `pg_statistic`(PostgreSQL) / `ANALYZE TABLE`(MySQL) 의 분포 추정에 의존. 대량 INSERT/배치 후 `ANALYZE` 를 직접 실행해 계획 퇴화를 막는다.
- **VACUUM(PostgreSQL)**: MVCC로 갱신/삭제된 행은 dead tuple로 남는다. autovacuum이 정리하지만 갱신이 폭증하면 bloat가 쌓인다. `pg_stat_user_tables.n_dead_tup` 모니터링, 심하면 `VACUUM (ANALYZE)`.

### 슬로우 쿼리 로그

- PostgreSQL: `log_min_duration_statement = 200ms` 로 임계 초과 쿼리 기록. `pg_stat_statements` 확장으로 총 시간/호출 수 기준 상위 쿼리 집계.
- MySQL: `slow_query_log = ON` + `long_query_time`, `performance_schema` / `sys.statement_analysis`.
- 원칙: 가장 느린 1개가 아니라 **총 누적 시간(빈도 x 단건 시간)** 이 큰 쿼리부터 고친다.

### 캐시 계층

- **DB 버퍼**: PostgreSQL `shared_buffers`(보통 RAM의 25% 부근), MySQL InnoDB `innodb_buffer_pool_size`. 핫 데이터가 메모리에 상주하면 디스크 I/O가 사라진다.
- **OS 페이지 캐시**: DB 버퍼 미스 시 운영체제 캐시가 다시 막아준다.
- **애플리케이션 캐시**: Redis/Memcached로 반복 조회를 흡수. 무효화 전략(TTL/이벤트)을 함께 설계.

## 4. 적용 / 함정

- **추측 튜닝 금지**: EXPLAIN/슬로우 로그로 병목을 측정하기 전에 인덱스를 추가하지 않는다. 안 쓰는 인덱스만 늘어난다.
- **`SELECT *`**: 불필요한 컬럼은 커버링 인덱스를 무력화하고 네트워크/메모리를 낭비한다. 필요한 컬럼만 명시.
- **OFFSET 페이지네이션**: 큰 OFFSET은 앞 행을 모두 스캔한다. 키셋 페이지네이션(`WHERE id < ? ORDER BY id DESC LIMIT n`)으로 전환.
- **단건 측정 함정**: 첫 실행은 캐시 미스로 느리다. 워밍업 후 반복 측정.
- **인덱스 추가만으로 해결 가정**: 낮은 선택도/잘못된 JOIN/통계 문제는 인덱스로 안 풀린다. 계획부터 본다.
- **autovacuum 끄기**: 일시적 부하 회피로 끄면 bloat가 누적되어 더 큰 장애로 돌아온다.

## 5. 관련

- [[../database-design/database-design|DB 설계 실무]] — 정규화/인덱스 종류/마이그레이션 등 설계 기반
- [[../postgresql/postgresql|PostgreSQL]] / [[../mysql/mysql|MySQL]] — 엔진별 통계/VACUUM/버퍼 풀 구현
- [[../../../30-resources/computer-science/database-theory/database-theory|database-theory]] — 비용 기반 옵티마이저/실행 계획 이론
- [[../database|↑ database hub]]
- 외부 참고: PostgreSQL 공식 문서(Using EXPLAIN, Routine Vacuuming, pg_stat_statements), MySQL 8 Optimization 가이드, Use The Index, Luke (use-the-index-luke.com)
