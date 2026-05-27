---
title: "troubleshooting · Redis OOM — maxmemory 도달"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [database, redis, cache, memory, eviction]
home_hub: "60-troubleshooting"
related: ["[[database]]"]
---

# Redis `OOM command not allowed`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 |

## 증상

```
OOM command not allowed when used memory > 'maxmemory'.
```

- `INFO memory` 의 `used_memory` 가 `maxmemory` 도달
- write command 거절, read 만 가능
- 특정 hot key / 큰 value 가 메모리 점유

## 원인

1. `maxmemory-policy` 가 `noeviction` (default — 거절)
2. TTL 없이 key 무한 누적
3. 큰 value (list / hash / stream) 가 GB 단위
4. fragmentation (mem_fragmentation_ratio > 1.5)

## 해결

### Eviction policy 설정
```
maxmemory 4gb
maxmemory-policy allkeys-lru
```

옵션:
- `noeviction` — default, 거절 (안전 우선)
- `allkeys-lru` — 전체에서 LRU 제거 (cache 용)
- `volatile-lru` — TTL 있는 key 만 LRU
- `allkeys-lfu` — LFU (자주 쓰는 것 보존)
- `volatile-ttl` — 짧은 TTL 먼저 제거

→ session / temp cache 는 `allkeys-lru`. 영속 데이터는 `noeviction` 유지 + 큰 key 정리.

### TTL 의무화
```java
// Bad
redis.opsForValue().set(key, value);

// Good
redis.opsForValue().set(key, value, Duration.ofHours(1));
```

```bash
# TTL 없는 key 찾기
redis-cli --scan --pattern '*' | xargs -L 100 redis-cli TTL | grep '^-1$' | wc -l
```

### 큰 key 분석
```bash
redis-cli --bigkeys
redis-cli --memkeys --memkeys-samples 1000

# 또는 RDB dump 분석
redis-rdb-tools dump.rdb -c memory | sort -k4 -n -r | head
```

### Fragmentation 해소
```bash
redis-cli MEMORY PURGE
# 또는 재시작 + load
```

```
mem_fragmentation_ratio: 1.05  # 좋음
mem_fragmentation_ratio: 1.8   # fragmented → restart 고려
```

### Hot key 분리
```
user:profile:42 → 5MB hash (모든 history)
↓
user:profile:42        → 기본 정보만
user:profile:42:history → 별도 (또는 DB 로)
```

## 진단

```bash
redis-cli INFO memory
redis-cli INFO clients
redis-cli MEMORY USAGE <key>
redis-cli OBJECT ENCODING <key>   # ziplist / hashtable
redis-cli SLOWLOG GET 10
redis-cli CLIENT LIST | grep "cmd=" | sort | uniq -c | sort -rn | head
```

## 재발 방지

- Redis 는 cache 로만 사용 (영속 데이터는 DB) — 의도 분리
- 모든 SET 에 TTL 의무 (lint 룰)
- 모니터링:
  ```
  - alert: RedisMemoryHigh
    expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.85
  ```
- maxmemory 는 OS RAM 의 60-70% (replica + COW + fragmentation 여유)

## 관련

- [[database]] / [[../backend/oom-heap]]
