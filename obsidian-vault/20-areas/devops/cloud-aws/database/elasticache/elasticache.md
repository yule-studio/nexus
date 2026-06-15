---
title: "Amazon ElastiCache"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T10:40:00+09:00
tags:
  - aws
  - database
  - cache
  - redis
  - memcached
---

# Amazon ElastiCache

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | ElastiCache hub |

**[[../database|↑ Database]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

매니지드 **Redis / Memcached**. 인메모리 캐시 / 세션 / pub-sub / 분산 락의 표준.

---

## 2. 왜 / 언제

- **캐시** — DB 부하 ↓, 응답 latency 마이크로초
- **세션 store** — 여러 응용 인스턴스가 공유
- **rate limit / counter / leaderboard** — atomic increment
- **분산 락** — `SET NX EX`
- **pub/sub** — Redis Streams / 채널
- **Geospatial / HyperLogLog / Bitmaps** — Redis 자료구조

> "Redis 를 직접 운영하기 싫을 때." Multi-AZ failover / patch / backup 매니지드.

---

## 3. 엔진 / 모드

| 엔진 | 의미 |
| --- | --- |
| **Redis (Cluster mode disabled)** | 단일 primary + 0-5 replica (read scale) |
| **Redis (Cluster mode enabled)** | 데이터 sharding (최대 500 노드, 수 TB) |
| **Memcached** | 단순 KV, 멀티스레드, persistence X |
| **Valkey** | Redis 7 fork (라이선스 이슈 후 OSS 진영) — ElastiCache 도 지원 |

→ 신규 = **Redis** (또는 Valkey). Memcached 는 단순 캐시 한정.

---

## 4. ElastiCache vs MemoryDB

| | ElastiCache for Redis | MemoryDB for Redis |
| --- | --- | --- |
| 용도 | **캐시** (in-memory, replica) | **primary store** (멀티 AZ 영속) |
| 지속성 | snapshot + AOF (옵션) | **transaction log → multi-AZ** |
| latency | μs read, μs write | μs read, **ms write** |
| 비용 | 보통 | 비쌈 |

→ 캐시 = ElastiCache. **DB 로 쓰려면** MemoryDB.

---

## 5. 표준 패턴

### 5.1 Cache-Aside (Lazy Loading)

```python
val = cache.get(key)
if val is None:
    val = db.query(...)
    cache.setex(key, 300, val)   # TTL 5분
return val
```

→ 가장 일반적. TTL 필수.

### 5.2 Write-Through
DB 쓰기와 동시에 캐시도 갱신. invalidation 문제 적음, 쓰기 latency ↑.

### 5.3 Cache stampede 방지
- jitter (`TTL ± random`)
- single-flight (분산 락)
- stale-while-revalidate

---

## 6. 비용 / 함정

- **노드 가격** — `cache.t4g.micro` ~$12/월부터, prod 은 `cache.r7g.large` 이상.
- **Multi-AZ** — primary + replica = 2배.
- **Network** — cross-AZ 트래픽 비쌈 (응용을 같은 AZ 에).
- **maxmemory-policy** — `allkeys-lru` 가 일반 (default `noeviction` 은 OOM).
- **persistence ON 시 fork 메모리 spike** — 메모리 50% 여유 권장.
- **key 디자인** — `user:{id}:profile` 같이 prefix.
- **N+1 호출** — pipeline / MGET 사용.

---

## 7. 관련

- [[../database|↑ Database]]
- [[../../../database/database|↗ Redis 자체 (20-areas)]]
- [[../rds/rds]] — primary DB
- [[../../security/secrets-manager/secrets-manager]] — Redis AUTH token 보관
