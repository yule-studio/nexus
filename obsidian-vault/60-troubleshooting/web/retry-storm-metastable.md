---
title: "troubleshooting · 재시도 폭풍 / 메타스테이블 장애 (retry storm)"
kind: troubleshooting
status: current
created_at: 2026-06-15
tags: [web, retry, backoff, circuit-breaker, resilience, metastable]
home_hub: "60-troubleshooting"
related: ["[[web]]", "[[cache-stampede]]", "[[../backend/thread-pool-exhausted]]"]
---

# 재시도 폭풍 / 메타스테이블 장애 (retry storm)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-06-15 | web | 최초 — 증폭/메타스테이블/백오프·서킷브레이커 |

**[[web|↑ web]]**

## 증상

- 부분 정전 / 한 AZ 지연 / DB 슬로우쿼리 같은 **일시적 트리거**가 발생했다가 사라졌는데도 시스템이 스스로 회복 못 하고 **과부하에 고착**.
- 들어오는 요청량(QPS)은 평소와 같거나 더 적은데 다운스트림(DB / 내부 API / 캐시)은 계속 100% 포화.
- 무중단처럼 보이던 서비스가 부분 장애에서 **전면 장애**로 번짐. 롤백을 해도, 트래픽을 일부 줄여도 안 풀림.
- 그래프 모양: 에러율과 다운스트림 latency 가 트리거 종료 후에도 평평하게 **높은 상태로 유지**(자가 회복선이 안 보임).

```
원인 트래픽 ─┐
            ▼  (트리거)
  정상 평형 ──────● ───────► 나쁜 평형(고착)
                  ▲
            트리거 사라져도 ────── 여기서 안 돌아옴
```

## 가설

- "들어오는 트래픽이 늘었나?" → LB 로그 보면 **유입 QPS 는 정상**. 외부 부하 증가 아님.
- "코드 배포가 원인인가?" → 배포 롤백해도 안 풀림. 코드 결함이 아니라 **부하 동역학** 문제.
- "용량(replica)을 늘리면?" → 늘리는 순간 잠깐 좋아졌다가 같은 평형으로 다시 빠짐 — 전형적 **메타스테이블** 신호.

## 원인 (root cause + 메커니즘)

다운스트림이 느려지면 클라이언트가 **재시도**를 한다. 그 결과:

```
다운스트림 느려짐
   → 클라이언트 타임아웃
   → 재시도 (유효 부하 ×2~3 증폭)
   → 다운스트림 더 포화
   → 더 많은 타임아웃
   → 더 많은 재시도  ⟲  (양의 피드백 루프)
```

이게 **메타스테이블 실패(metastable failure)** 다. 시스템에 두 개의 평형이 존재한다 — 정상 평형과 나쁜 평형. 트리거가 시스템을 나쁜 평형으로 한 번 밀어 넣으면, **트리거가 사라져도 재시도 부하 자체가 장애를 지속**시킨다. 즉 원인(트리거)과 지속요인(재시도 증폭)이 분리돼 있어서, 원인을 제거하는 통상적 대응이 안 먹힌다.

- **유효 부하(work amplification)**: 한 요청이 3회 재시도되면 다운스트림 입장에서 1 요청이 4 요청이 된다. 여러 계층이 각자 재시도하면 **곱셈**으로 증폭된다(L1 3회 × L2 3회 = 9배). 이게 retry amplification.
- 캐시가 같이 무너지면 [[cache-stampede]] 와 겹쳐 부하가 한 번 더 폭증한다.
- thread / connection pool 이 재시도로 가득 차면 [[../backend/thread-pool-exhausted]] 로 직결된다.

실제 사례로 짧게: **AWS S3 2017-02-28 us-east-1 대장애** — 운영자가 디버깅 명령을 입력하다 오타로 **의도보다 많은 서버를 제거**, 서브시스템이 안전 용량 아래로 떨어졌다. 이후 서브시스템 재시작 + 그동안 쌓인 요청의 재요청 폭증이 회복을 방해해 수 시간 장기화됐다. "용량 부족 + 재시도/재시작 부하"가 어떻게 self-recovery 를 막는지의 교과서 사례다. (대규모 분산에서 retry amplification 은 단골 원인이다.)

## 해결

### 1. 지수 백오프 + 지터(full jitter)

고정 간격 / 동기화된 재시도는 금지 — 모든 클라이언트가 같은 순간에 다시 때려 **재시도 thundering herd** 를 만든다.

```python
# Bad — 동기화된 고정 backoff (모두 같은 시점에 재시도)
sleep = base * (2 ** attempt)

# Good — full jitter (재시도 시점을 무작위로 흩뿌림)
import random
sleep = random.uniform(0, base * (2 ** attempt))
```

```java
// resilience4j IntervalFunction — 지수 backoff + 무작위화
IntervalFunction fn =
    IntervalFunction.ofExponentialRandomBackoff(
        Duration.ofMillis(100), /* multiplier */ 2.0, /* randomization */ 0.5);
RetryConfig cfg = RetryConfig.custom()
    .maxAttempts(3)               // 무한 재시도 금지
    .intervalFunction(fn)
    .build();
```

### 2. 재시도 예산(retry budget) / 토큰 버킷

전체 요청 대비 재시도 비율에 **상한**(예: 10%)을 둔다. 예산이 바닥나면 재시도를 포기하고 바로 실패시킨다 — 증폭의 곱셈을 끊는 핵심 장치.

```yaml
# Envoy retry budget 개념
retry_budget:
  budget_percent: 10      # 활성 요청의 10% 까지만 재시도 허용
  min_retry_concurrency: 3
```

### 3. 서킷 브레이커

실패율이 임계를 넘으면 빠르게 **open** → 호출을 즉시 차단해 다운스트림에 숨 쉴 틈을 준다 → 일정 시간 뒤 **half-open** 탐침으로 회복 확인.

```yaml
resilience4j.circuitbreaker.instances.downstream:
  slidingWindowSize: 100
  failureRateThreshold: 50      # 실패율 50% 초과 시 open
  waitDurationInOpenState: 10s  # open 유지 → 회복 시간 확보
  permittedNumberOfCallsInHalfOpenState: 5
```

### 4. 부하 차단(load shedding) + 멱등성

- 포화 시점에 **우선순위 큐**로 저우선 요청을 먼저 버린다(429 / 503 빠른 반환).
- 타임아웃은 **짧고 명확하게** — 무한 대기는 재시도 폭풍의 연료.
- 재시도는 **멱등(idempotent)** 한 연산에만 안전하게 적용한다 → [[../backend/payment-webhook-idempotency]] 의 멱등키 패턴 필수.

### 5. 계층별 재시도 단계 제한

L1·L2·L3 가 각자 재시도하면 곱셈 증폭. **재시도는 한 계층에서만** 하고(보통 가장 바깥 / 클라이언트 가까운 쪽), 나머지는 fail-fast.

## 핵심 방어 비교 — 백오프+지터 vs 서킷 브레이커 (4구조)

이 둘은 경쟁이 아니라 **상보적**이다.

1. **왜 필요한가** — 지터는 재시도들이 같은 순간 몰리는 **동기화를 깨서** 순간 부하 스파이크를 평탄화한다. 서킷 브레이커는 다운스트림이 이미 죽었을 때 호출 자체를 끊어 **회복 시간을 벌어준다**. 하나는 "언제 다시 보낼지", 다른 하나는 "지금 보낼지 말지"를 다룬다.
2. **안 하면 무슨 문제** — 지터만 있고 브레이커가 없으면, 흩뿌려진 재시도라도 죽은 다운스트림을 계속 두드려 회복 자체를 막는다(메타스테이블 고착). 브레이커만 있고 지터가 없으면, half-open 으로 회로가 닫히는 순간 대기하던 모든 클라이언트가 **동시에** 쏟아져 곧바로 다시 open — 회로가 open↔closed 를 진동(flapping)한다.
3. **대안** — 재시도 예산(2번)으로 증폭을 비율로 직접 막거나, deadline propagation(상위 타임아웃을 하위로 전파해 의미 없는 재시도를 원천 차단)으로 대체/보강할 수 있다. 큐 기반 비동기 처리로 재시도를 워커 쪽에 흡수시키는 설계도 있다.
4. **트레이드오프** — 지터는 평균 응답 latency 를 약간 늘린다(무작위 대기). 서킷 브레이커는 **false-open** 위험(일시적 스파이크에 과민 반응해 멀쩡한 다운스트림 차단) + open 동안 모든 요청 즉시 실패라는 가용성 비용이 있다. 임계값(`failureRateThreshold`, `waitDuration`)을 너무 민감하게 잡으면 정상 트래픽을 끊고, 너무 둔하게 잡으면 장애를 못 막는다.

→ 실전 권장: **지터(동기화 깨기) + 브레이커(회복 시간) + 재시도 예산(증폭 상한)** 세 개를 함께 깐다.

## 진단

```promql
# 재시도 비율 (증폭 감지) — 0.1 넘으면 위험
sum(rate(http_client_retries_total[1m]))
  / sum(rate(http_client_requests_total[1m]))

# 서킷 상태 (0=closed 1=open 2=half_open)
resilience4j_circuitbreaker_state

# 다운스트림 포화 + 유입 QPS 분리해서 보기
rate(downstream_inflight_requests[1m])   # 포화는 올라가는데
rate(ingress_requests_total[1m])         # 유입은 평소 → metastable 의심
```

```bash
# 유효 부하 증폭 비율 = 다운스트림 수신 / LB 유입
echo "amplification = downstream_qps / ingress_qps"   # 1 이면 정상, 2~3 이면 retry storm
```

## 재발 방지

- **재시도 비율 / 서킷 상태 상시 모니터링** + 알람:
  ```yaml
  - alert: RetryAmplification
    expr: sum(rate(http_client_retries_total[5m]))
        / sum(rate(http_client_requests_total[5m])) > 0.2
    for: 2m
  ```
- **카오스 / 부하 테스트로 메타스테이블 재현**: toxiproxy 로 다운스트림에 지연 주입 → 트리거 제거 후 시스템이 스스로 정상 평형으로 돌아오는지 확인. 안 돌아오면 방어 설계 미흡.
- **멱등키 강제** — 재시도 가능한 모든 쓰기 경로에 멱등키 의무화 ([[../backend/payment-webhook-idempotency]]).
- **계층별 타임아웃 예산 일관성** — 상위 타임아웃이 하위 (타임아웃 × 재시도 횟수)보다 커야 의미 있음. 안 맞으면 상위가 먼저 끊겨 하위 재시도가 전부 낭비.
- **PR 룰**: 새 외부 호출 / 재시도 추가 시 backoff + jitter + 최대 횟수 + 멱등성 검토 의무.
- DNS / 외부 의존 실패도 재시도 폭풍의 트리거가 됨 → [[../network/dns-resolve-fail]] 경로 점검.

## 관련

- [[web]]
- [[cache-stampede]] — 캐시 붕괴가 retry storm 의 동반 트리거
- [[../backend/thread-pool-exhausted]] — 재시도가 thread/connection pool 을 고갈시킴
- [[../backend/payment-webhook-idempotency]] — 안전한 재시도의 전제(멱등성)
- [[../network/dns-resolve-fail]] — 외부 의존 실패 → 재시도 폭풍 트리거
- [[../troubleshooting|↑ 60-troubleshooting]]
- 책/참고: Brooker, "Metastable Failures in Distributed Systems"; AWS Builders' Library — Timeouts/Retries/Backoff with Jitter
