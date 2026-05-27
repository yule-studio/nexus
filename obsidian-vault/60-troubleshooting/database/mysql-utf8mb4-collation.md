---
title: "troubleshooting · MySQL utf8mb3 ↔ utf8mb4 collation 충돌"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [database, mysql, charset, collation, utf8mb4]
home_hub: "60-troubleshooting"
related: ["[[database]]"]
---

# MySQL `Illegal mix of collations`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 |

## 증상

```
ERROR 1267: Illegal mix of collations
  (utf8mb3_general_ci, IMPLICIT) and (utf8mb4_0900_ai_ci, IMPLICIT)
  for operation '='
```

- JOIN 시 발생
- 이모지 / 4-byte 문자 (𝕜, 🎉) 가 깨짐 ("???" 또는 truncate)
- MySQL 5.7 → 8.0 업그레이드 또는 새 table 추가 후

## 원인 (root cause)

- `utf8mb3` (옛 utf8) — 3-byte max, BMP 만. 이모지 / 일부 한자 깨짐
- `utf8mb4` — 4-byte max, 전체 Unicode 지원
- MySQL 5.7 default = utf8mb3, MySQL 8.0 default = utf8mb4_0900_ai_ci
- 같은 DB 안에 두 charset 혼재 → JOIN 비교 시 충돌

## 해결

### 전 컬럼 utf8mb4 통일
```sql
-- DB
ALTER DATABASE mydb CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;

-- Table
ALTER TABLE users CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;

-- 또는 컬럼별
ALTER TABLE users MODIFY COLUMN name VARCHAR(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
```

→ 큰 table 은 `pt-online-schema-change` 로.

### JDBC URL 명시
```
jdbc:mysql://host:3306/db?characterEncoding=UTF-8&useUnicode=true&connectionCollation=utf8mb4_0900_ai_ci
```

### Server config
```ini
[mysqld]
character-set-server = utf8mb4
collation-server = utf8mb4_0900_ai_ci
init_connect = 'SET NAMES utf8mb4'

[client]
default-character-set = utf8mb4
```

### Index 길이 함정 (MySQL 5.7)
utf8mb4 는 4 byte/char → VARCHAR(255) 인덱스 = 1020 byte (5.7 의 767 한도 초과):
```sql
-- 해결: innodb_large_prefix + DYNAMIC row format
SET GLOBAL innodb_large_prefix = ON;
ALTER TABLE users ROW_FORMAT=DYNAMIC;
```

MySQL 8+ 는 default OK.

### Collation 선택
- `utf8mb4_0900_ai_ci` — 8.0 default, accent/case insensitive (`á = a`)
- `utf8mb4_unicode_ci` — 5.7 호환
- `utf8mb4_bin` — case sensitive, 한글 정렬 자연

한글 정렬은 `utf8mb4_unicode_ci` 또는 `utf8mb4_ko_0900_as_cs` 선호.

## 진단

```sql
-- DB / table / column charset
SHOW VARIABLES LIKE '%character%';
SHOW VARIABLES LIKE '%collation%';

SELECT default_character_set_name, default_collation_name
FROM information_schema.schemata WHERE schema_name = 'mydb';

SELECT table_name, table_collation
FROM information_schema.tables WHERE table_schema = 'mydb';

SELECT table_name, column_name, character_set_name, collation_name
FROM information_schema.columns
WHERE table_schema = 'mydb' AND character_set_name IS NOT NULL;

-- 이모지 저장 가능 확인
INSERT INTO users (name) VALUES ('이모지 🎉');
SELECT name, HEX(name) FROM users WHERE name LIKE '%🎉%';
-- HEX = F09F8E89 → 4 byte 저장 OK
```

## 재발 방지

- 새 DB 는 처음부터 `utf8mb4_0900_ai_ci`
- migration script 에 charset / collation 명시 의무
- CI 단계 schema lint — utf8mb3 column 발견 시 fail

## 관련

- [[database]]
