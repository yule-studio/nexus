---
title: "Amazon MemoryDB for Redis"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-06-01T12:20:00+09:00
tags:
  - aws
  - database
  - redis
  - memorydb
---

# Amazon MemoryDB for Redis

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-01 | engineering-agent/tech-lead | MemoryDB hub |

**[[../database|↑ Database]]** · **[[../../cloud-aws|↑↑ AWS]]**

---

## 1. 한 줄

**Redis 호환 + 멀티 AZ 영속 트랜잭션 로그**. ElastiCache 가 "캐시" 라면 MemoryDB 는 "primary DB" — 인메모리 속도 + 영속성 둘 다.

## 2. 왜 / 언제

- Redis 를 단일 source of truth 로 (DynamoDB / RDS 대신)
- 마이크로초 latency 가 필수인 응용 (게임 / 광고 입찰 / 금융)
- Redis Streams / Pub-Sub / 데이터 구조 활용 + 데이터 손실 X

## 3. ElastiCache vs MemoryDB

| | ElastiCache for Redis | MemoryDB |
| --- | --- | --- |
| 역할 | 캐시 | **primary store** |
| 지속성 | AOF (옵션) | 멀티 AZ 트랜잭션 로그 |
| write latency | μs | **ms** (영속 보장 때문) |
| 비용 | 보통 | 비쌈 |

## 4. 함정

- 비쌈 — 캐시면 ElastiCache 가 적합
- write latency 가 μs 가 아니라 ms — 그래도 RDS 보다 훨씬 빠름

## 5. 관련

- [[../database|↑ Database]]
- [[../elasticache/elasticache]]
- [[../dynamodb/dynamodb]]
