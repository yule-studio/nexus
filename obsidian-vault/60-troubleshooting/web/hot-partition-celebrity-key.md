---
title: "troubleshooting · 핫 파티션 / 셀러브리티 키 (hot shard)"
kind: troubleshooting
status: current
created_at: 2026-06-15
tags: [web, sharding, hot-partition, dynamodb, scalability, celebrity-problem]
home_hub: "60-troubleshooting"
related: ["[[web]]", "[[cache-stampede]]"]
---

# 핫 파티션 / 셀러브리티 키 (hot shard)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-15 | web | 최초 — 부하 편향/키 솔팅/셀럽 fanout |

**[[web|↑ web]]**

## 1. 증상

- 클러스터 **전체 용량은 충분**한데 특정 노드/샤드 **하나만 100% 포화**, 나머지는 한가하다.
- 특정 키 / 사용자 / 상품에 대한 요청만 느리거나 throttle. 같은 테이블의 다른 키는 멀쩡.
- DynamoDB `ProvisionedThroughputExceededException` / throttle 이 **한 파티션에서만** 발생.
- Cassandra 한 노드만 CPU/디스크 핫스팟, Kafka **한 파티션만** consumer lag 이 계속 쌓임.
- p99 레이턴시가 **특정 엔터티에서만** 튄다. 평균/중앙값은 정상으로 보여 놓치기 쉽다.

```
# DynamoDB CloudWatch — 테이블 합계는 여유, 특정 파티션만 throttle
ConsumedWriteCapacity (table)   : 1,200 / 4,000 WCU   (30% — 여유 많음)
WriteThrottleEvents             : 8,500 / min          (← 폭증)
# 그런데 throttle 대상 키를 까보면 전부 동일:
ThrottledRequests by key        : post#9183(셀럽) 99.7%, 그 외 0.3%
```

- "용량 더 늘렸는데 그 키만 여전히 느림" — 총 용량 증설로 안 풀리는 게 핵심 신호.

## 2. 원인 (root cause + 메커니즘)

데이터/트래픽은 **파티션 키 기준으로 분산**된다(해시 또는 레인지). 한 키가 비대칭적으로
많은 부하를 받으면, 그 키가 속한 **단일 파티션/노드 하나가 병목**이 된다.

```
파티션 키 = hash(key) % N
  user#100 → P3   ┐
  user#101 → P7   ├─ 보통은 N개 파티션에 골고루 흩어짐
  user#102 → P1   ┘
  post#9183(셀럽) → P3 ──── 한 키가 전 트래픽의 80% → P3 만 불탐
                            (나머지 P0,P1,P2,P4... 는 한가)
```

- **균등 해시도 "키 1개가 핫"이면 못 푼다.** 같은 키는 항상 같은 파티션으로 가기 때문.
  파티션을 더 쪼개도(N 증가) 그 키는 여전히 한 파티션에만 들어간다.

전형 패턴:

- **셀러브리티 문제 (celebrity problem)** — 팔로워 수천만 명인 계정이 글을 올리면,
  그 글을 **모든 팔로워 타임라인에 쓰는(fanout-on-write)** 순간 쓰기가 폭증.
  트위터/SNS 의 고전. 일반 사용자(팔로워 수백) 와 부하가 4~6자릿수 차이.
- **바이럴 상품 / 이벤트** — 한정판 드롭, 인기 게시물의 **좋아요 카운터 단일 row**에
  쓰기가 집중(같은 row 갱신 경합 → 직렬화). 읽기뿐 아니라 쓰기 핫스팟.
- **단일 테넌트 편중** — 멀티테넌트에서 거대 고객 1곳(`tenant#bigcorp`)이 전 트래픽을
  독식. 작은 테넌트 수천 곳과 같은 파티션 전략을 공유하면 그 하나만 탄다.
- **시간 키 핫스팟** — 파티션 키가 `날짜`면 **오늘 날짜** 파티션에만 모든 쓰기가 몰린다
  (어제는 0, 오늘만 100%). 순증 ID(auto-increment)도 항상 끝 파티션에 쏠림.

## 3. 해결

### 3.1 키 솔팅 / 쓰기 분산 (write sharding)

핫 키에 **샤드 접미사**를 붙여 N개 파티션에 흩고, 읽을 때 scatter-gather 로 합산.

```python
import random

SHARDS = 10  # 핫 키 1개를 10개 논리 샤드로 분산

# 쓰기: 랜덤 샤드에 흩뿌림 → P3 한 곳이 아니라 10개 파티션으로 분산
def write_like(post_id, user_id):
    shard = random.randint(0, SHARDS - 1)
    pk = f"likes#{post_id}#{shard}"      # likes#9183#0 .. likes#9183#9
    dynamo.update_item(Key={"pk": pk}, AddCount=1)

# 읽기: 모든 샤드를 모아 합산 (scatter-gather)
def read_like_count(post_id):
    total = 0
    for shard in range(SHARDS):
        item = dynamo.get_item(Key={"pk": f"likes#{post_id}#{shard}"})
        total += item.get("count", 0)
    return total                          # N번 읽고 합치는 비용이 트레이드오프
```

- **샤딩된 카운터(sharded counter)** — 단일 카운터 row 경합을 N개 row 로 쪼개 경합 제거.
  읽을 때만 합산. 좋아요/조회수처럼 쓰기 헤비 카운터의 표준 해법.

### 3.2 읽기 핫 키 캐싱

읽기 비중이 큰 핫 키는 **캐시(Redis / CDN)로 흡수**해 원본 파티션을 보호한다.

```python
def read_post(post_id):
    v = redis.get(f"post:{post_id}")
    if v: return v                        # 핫 키 트래픽이 원본까지 안 감
    v = dynamo.get_item(Key={"pk": f"post#{post_id}"})
    redis.set(f"post:{post_id}", v, ex=60)
    return v
```

- 핫 키는 **캐시 스탬피드도 잘 난다** — 만료 순간 동시 miss 가 그 한 파티션을 때린다.
  단일 비행/조기 만료/지터를 같이 깔 것 → [[cache-stampede]].

### 3.3 셀럽은 별도 전략 (하이브리드 fanout)

쓰기 분산만으로는 셀럽의 거대 fanout 자체를 못 줄인다. **읽기 시점으로 미룬다.**

- **일반 사용자** → `fanout-on-write` (올릴 때 팔로워 타임라인에 미리 써둠, 읽기 빠름).
- **셀럽** → `fanout-on-read` (셀럽 글은 미리 안 쓰고, 팔로워가 **타임라인을 읽을 때
  합쳐서** 가져옴). 수천만 건 쓰기 폭발을 회피.

```
타임라인 조회 = (미리 쓰여진 일반 사용자 글)  ∪  (읽을 때 합치는 셀럽 글)
                 ↑ fanout-on-write             ↑ fanout-on-read (셀럽만)
```

### 3.4 평탄화 / 격리

- **요청 합치기(coalescing)** — 같은 핫 엔터티 in-flight 요청을 묶어 원본 호출 1회로.
- **큐로 평탄화** — 좋아요 같은 쓰기를 큐에 받아 배치로 흡수, 피크를 시간축에 분산.
- **셀 기반 격리** — 핫 엔터티 전용 용량/노드(cell)로 분리해 blast radius 를 가둔다.

### 3.5 DynamoDB adaptive capacity 의 한계

DynamoDB 의 adaptive capacity 가 핫 파티션을 **일부** 흡수하지만,
**단일 파티션 처리량 상한**(파티션당 WCU/RCU 한도, 항목당 한도)은 못 넘는다.
근본 해결은 **키 설계**다 — adaptive 에 의존해 셀럽 키를 그대로 두면 결국 throttle.

## 4. 핵심 방어 — 키 솔팅(쓰기 분산) vs 캐싱(읽기 흡수) (왜 / 트레이드오프)

둘 다 핫 키 부하를 푼다. 하지만 **읽기 핫이냐 쓰기 핫이냐**로 선택이 갈린다.

1. **왜 필요한가**
   - 캐싱: 읽기 헤비 핫 키는 캐시 한 층이면 원본 파티션 트래픽이 0 에 수렴.
     **간단하고 즉효** — 코드 변경 최소, 키 구조 그대로.
   - 키 솔팅: 쓰기 헤비 핫 키는 캐시가 못 막는다(쓰기는 원본에 도달해야 함).
     부하 자체를 **N개 파티션으로 물리적으로 쪼개야** 단일 파티션 한도를 넘는다.

2. **안 하면 무엇이 터지나**
   - 읽기 핫에 캐싱 안 하면 → 그 한 파티션이 RCU 한도에서 throttle, 핫 키만 느려짐.
   - 쓰기 핫에 캐싱만 하면 → **안 먹힌다.** 좋아요 폭주는 캐시를 지나 원본 단일 row
     경합으로 직행, WriteThrottle 폭증 + 재시도 → [[retry-storm-metastable]].

3. **대안 / 언제 무엇을**
   - **읽기 헤비 핫 키(인기 게시물 조회)** → 캐싱. 쓰기 분산은 과한 복잡도.
   - **쓰기 헤비 핫 키(좋아요 카운터, 셀럽 fanout)** → 키 솔팅 / 샤딩 카운터.
     캐시로는 원천적으로 해결 불가.
   - 둘 다 핫(인기 글의 조회+좋아요)이면 **캐싱 + 샤딩 카운터를 함께** 깐다.

4. **트레이드오프**
   - 캐싱: 싸고 빠르지만 **일관성/만료**를 떠안고, 핫 키 만료 순간 stampede 위험.
   - 키 솔팅: 단일 파티션 한도를 넘지만 **읽기 때 scatter-gather(N번 읽고 합산)**
     비용과, **정렬/범위 쿼리 손실**(샤드 접미사로 키가 흩어져 단일 인덱스 정렬 불가)을
     감수한다. 샤드 수 N 도 운영 파라미터 — 작으면 분산 부족, 크면 읽기 비용 증가.

→ 실무 권장: **읽기 핫은 캐싱, 쓰기 핫은 솔팅**으로 1차 분류하고, 셀럽처럼
   fanout 이 큰 쓰기는 **하이브리드 fanout** 으로 부하 발생 자체를 미룬다.

## 5. 재발 방지

- **파티션별 메트릭** — 테이블 합계가 아니라 **파티션당 throughput / throttle,
  노드별 부하 편차**를 본다. 합계만 보면 한 파티션 포화를 놓친다.
  ```
  - alert: HotPartitionThrottle
    # 테이블 용량은 여유인데 throttle 이 오르면 = 핫 파티션
    expr: WriteThrottleEvents > 0 and ConsumedWCU < 0.6 * ProvisionedWCU
  ```
- **파티션 키 설계 리뷰** — 신규 테이블/토픽은 키의 **카디널리티·균등성·핫스팟 가능성**을
  리뷰 게이트로. 날짜/순증 ID/단일 테넌트를 파티션 키로 쓰면 자동 경고.
- **셀럽 / 바이럴 시나리오 부하 테스트** — 단일 핫 키에 트래픽을 몰아 단일 파티션
  포화를 실제로 재현하고, 솔팅/하이브리드 fanout 적용 전후 throttle 을 비교.
- **핫 키 자동 감지 + 캐시 승격** — 키별 QPS 를 추적해 임계 초과 키를 자동으로 캐시
  승격하거나 샤딩 대상으로 표면화.

## 6. 관련

- [[web]] · [[cache-stampede]] (핫 키 만료 순간 동시 miss — 핫 파티션을 또 때림)
- [[retry-storm-metastable]] (throttle 후 일제 재시도가 같은 파티션을 증폭)
- [[../database/redis-oom]] (핫 키를 캐시로 흡수 → 캐시 메모리 압박으로 번짐)
- [[../database/slow-query]] (핫 row 경합으로 그 키 쿼리만 느려짐)
- [[../troubleshooting|↑ 60-troubleshooting]]
