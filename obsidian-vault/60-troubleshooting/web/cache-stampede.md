---
title: "troubleshooting · 캐시 스탬피드 (thundering herd / dogpile)"
kind: troubleshooting
status: current
created_at: 2026-06-15
tags: [web, cache, redis, thundering-herd, performance, scalability]
home_hub: "60-troubleshooting"
related: ["[[web]]", "[[../database/redis-oom]]"]
---

# 캐시 스탬피드 (thundering herd / dogpile)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-15 | web | 최초 — 원리/단일비행/조기만료/SWR |

**[[web|↑ web]]**

## 1. 증상

- 트래픽은 평소와 비슷한데 **특정 시점에만** DB / 원본 부하가 수직으로 튄다.
- 그 시점이 인기 캐시 키의 **TTL 만료 순간**과 정확히 일치.
- 응답 지연 → 타임아웃 → upstream 큐 적체 → 연쇄 장애.
- 로그에 **동일한 쿼리/원본 호출이 수백~수천 개 동시** 발생.

```
12:00:00.000  GET product:hot:42  → cache miss → SELECT ... (1)
12:00:00.001  GET product:hot:42  → cache miss → SELECT ... (2)
12:00:00.001  GET product:hot:42  → cache miss → SELECT ... (3)
...           (같은 ms 안에 수천 건, 전부 같은 SELECT)
```

- 재시작 / 배포 직후 **콜드 캐시** 상태에서 특히 심함 (전 키가 동시에 miss).

## 2. 원인 (root cause + 메커니즘)

핫 키 하나가 만료되는 순간, 그 직후 도착한 N개의 동시 요청이 **전부 캐시 미스**가 된다.
미스난 요청은 전부 원본(DB)으로 직행한다 → **thundering herd(dogpile)**.

```
TTL 만료 ──┐
           │  [요청1] miss → DB ┐
           │  [요청2] miss → DB ├─ 한 키의 재계산을 N번 중복 → DB 과부하
           │  [요청N] miss → DB ┘
           ▼
        DB 느려짐 → 응답 더 늦음 → 그동안 캐시 채워지지 않음
           → 미스 요청 더 쌓임 → 악순환(metastable)
```

- 정상 상태에서는 캐시가 DB를 가려주지만, 만료 **틈** 동안에는 가림막이 사라진다.
- 한 번 재계산하면 끝날 일을 N번 반복하는 것이 본질 — **중복 작업** 문제.
- 대규모 트래픽 사이트가 반드시 만나는 고전 문제. 페이스북 memcache 의
  **leases** 논문, 레딧 / 뉴스 사이트의 핫 페이지가 대표 사례.

변형: 여러 키를 **같은 TTL**로 동시에 채우면(예: 배포 직후 일괄 워밍업) 그 키들이
한꺼번에 만료된다 → **synchronized expiry**. 만료 폭발이 주기적으로 반복된다.

## 3. 해결

### 3.1 단일 비행 (single-flight) / 락

한 키에 대해 **단 하나의 요청만** 원본을 재계산하고, 나머지는 대기했다가 그 결과를 공유.

```go
// Go: golang.org/x/sync/singleflight
var g singleflight.Group

func GetProduct(id string) (*Product, error) {
    v, err, _ := g.Do("product:"+id, func() (any, error) {
        return loadFromDB(id) // 동시 N건이 와도 실제 호출은 1번
    })
    return v.(*Product), err
}
```

```bash
# Redis 분산 락 (멀티 인스턴스): SET NX 로 재계산 소유권 1개만
SET lock:product:42 <token> NX PX 3000
# 획득 실패한 요청 → 짧게 sleep 후 캐시 재조회(이미 채워졌을 확률 높음)
# 작업 끝나면 토큰 일치 확인 후 DEL (Lua) — 남의 락 해제 방지
```

### 3.2 조기 확률적 만료 (probabilistic early recomputation, XFetch)

TTL이 끝나기 **직전** 일부 요청이 확률적으로 미리 갱신 → 동시 만료 회피.

```python
# delta = 직전 재계산에 걸린 시간, beta ≈ 1.0
# 만료 임박 + 무거운 재계산일수록 일찍 갱신될 확률↑
if now - delta * beta * math.log(random.random()) >= expiry:
    value = recompute()        # 한두 요청만 선제 갱신
    cache.set(key, value, ttl, meta_delta=measured_delta)
else:
    return value               # 나머지는 옛 값 그대로 반환
```

락이 없어 대기가 없고, 만료 직전 단 몇 요청만 갱신을 떠안는다.

### 3.3 stale-while-revalidate (SWR)

만료돼도 **옛 값을 즉시 반환**하고, 갱신은 백그라운드로 돌린다.

```
Cache-Control: max-age=60, stale-while-revalidate=30
```

```python
# 앱 레벨
entry = cache.get(key)
if entry and entry.fresh:
    return entry.value
if entry:                      # stale 하지만 존재
    enqueue_refresh(key)       # 비동기 갱신
    return entry.value         # 사용자는 기다리지 않음
return recompute_blocking(key) # 진짜 없을 때만 블록
```

### 3.4 TTL 지터 (jitter)

TTL에 랜덤 가감으로 동시 만료를 시간축에 흩뿌린다.

```python
import random
base, spread = 300, 60
cache.set(key, value, ttl=base + random.randint(-spread, spread))  # 300s ± 60s
```

synchronized expiry(3.2절 변형)에 대한 가장 싸고 효과 큰 방어.

### 3.5 기타

- **요청 합치기(coalescing)**: 같은 키 in-flight 요청을 하나로 묶어 결과 fan-out.
- **음수 캐싱(negative caching)**: "없음" 결과도 짧은 TTL로 캐싱 → 존재하지 않는
  키로 매번 DB 때리는 것 방지.

## 4. 핵심 방어 — 단일 비행 vs 조기 만료 (왜 / 트레이드오프)

같은 herd를 막지만 성격이 다르다. 두 방어를 4구조로 비교한다.

1. **왜 필요한가**
   - 단일 비행: 만료 틈에서 **원본 재계산을 1회로 강제**해 DB가 받는 중복 부하를 산술적으로 제거.
   - 조기 만료: 애초에 **다 같이 miss나는 순간**을 만들지 않아 herd가 형성되기 전에 분산.

2. **안 하면 무엇이 터지나**
   - 둘 다 없으면 핫 키 만료 = N중 재계산. DB QPS가 캐시 미스 폭만큼 곱연산으로 튀어
     타임아웃 → metastable 장애([[retry-storm-metastable]]와 동일한 증폭 고리).

3. **대안 / 언제 무엇을**
   - **강한 일관성·재계산이 매우 비쌈(무거운 집계, 외부 API 과금)** → 단일 비행/락.
     중복 호출을 1로 떨어뜨리는 보장이 필요할 때.
   - **수많은 핫 키·약간의 staleness 허용·락 운영 부담 회피** → 조기 만료(XFetch) / SWR.
     부드럽고 대기가 없어 tail latency 에 유리.

4. **트레이드오프**
   - 단일 비행: 보장은 강하지만 **대기 요청의 latency**가 재계산 시간에 묶이고,
     분산 락은 만료/리스 갱신/소유권 검증 같은 **락 관리 복잡도와 장애 모드**(락 보유자
     사망 시 멈춤)를 새로 들인다.
   - 조기 만료: 락이 없어 부드럽지만 **약간의 추가 계산**(만료 전 선제 갱신)과
     **약간의 staleness**를 감수한다. 또 확률 파라미터(beta)를 잘못 잡으면 너무 자주/늦게 갱신.

→ 실무 권장: **TTL 지터를 기본**으로 깔고, 핫 키에 **단일 비행 또는 조기 만료**를
   덧대고, 사용자 응답 경로에는 **SWR**로 latency를 끊는다. 세 층은 상호 보완적이다.

## 5. 재발 방지

- **핫 키 식별 / 모니터링**: 캐시 미스율, 키별 원본 QPS, 만료 시각 분포를 본다.
  미스율 스파이크가 만료 시각과 겹치면 stampede 신호.
  ```
  - alert: OriginQPSSpikeOnMiss
    expr: rate(db_query_total[1m]) > 5 * rate(db_query_total[10m] offset 10m)
  ```
- **콜드 스타트 워밍업**: 배포/재시작 직후 핫 키를 순차(지터 포함) 프리로드 →
  콜드 캐시 동시 miss 폭발 방지.
- **TTL 지터 기본화**: 일괄 SET 경로에 jitter 미적용 시 lint / 리뷰에서 차단.
- **부하 테스트로 재현**: 단일 핫 키를 TTL 만료 직전 동시 N요청으로 때려
  만료 동시성을 실제로 재현하고, 방어 적용 전/후 DB QPS 스파이크를 비교.

## 6. 관련

- [[web]] · [[../database/redis-oom]] (만료 회피한다고 TTL 늘리면 메모리 압박으로 번짐)
- [[retry-storm-metastable]] (부하 증폭 연쇄 — stampede가 트리거가 됨)
- [[../backend/thread-pool-exhausted]] (느려진 원본이 워커를 소진)
- [[../troubleshooting|↑ 60-troubleshooting]]
