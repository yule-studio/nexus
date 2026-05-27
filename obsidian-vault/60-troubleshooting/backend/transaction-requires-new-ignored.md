---
title: "troubleshooting · @Transactional(REQUIRES_NEW) 가 무시됨"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [backend, spring, transaction, aop, propagation]
home_hub: "60-troubleshooting"
related:
  - "[[backend]]"
  - "[[aop-self-invocation]]"
---

# @Transactional(REQUIRES_NEW) 가 무시됨

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 — self-invocation / proxy AOP / 빈 분리 |

## 증상

- 의도: 부분 작업을 별도 트랜잭션으로 분리 → 부모 트랜잭션이 rollback 돼도 자식은 commit
- 실제: 자식 메서드도 부모 트랜잭션에 합류 → 같이 rollback
- 로그 audit / event publish 가 비즈니스 실패 시 함께 사라짐

```java
@Service
public class OrderService {

    @Transactional
    public void process(Order o) {
        save(o);
        logHistory(o);          // ← REQUIRES_NEW 인데 무시됨
        if (failBusiness()) throw new BizException();  // → logHistory 도 rollback
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logHistory(Order o) { historyRepo.save(...); }
}
```

## 가설

- `@Transactional` annotation 이 작동 안 하나 → 다른 메서드는 정상
- propagation 설정 오타? → 정확함
- 실제: Spring AOP proxy 가 **self-invocation** 을 가로채지 못함

## 원인 (root cause)

Spring `@Transactional` 은 AOP proxy 기반. proxy 는 외부 호출만 가로챌 수 있음.

```
외부 caller → [proxy] → OrderService.process()
                          ↓ this.logHistory()   ← proxy 거치지 않음
                            (그냥 같은 객체 메서드 호출)
```

→ `logHistory` 의 `@Transactional` 메타데이터 무시. 부모 트랜잭션에서 그대로 실행.

같은 원리로 `@Async` / `@Cacheable` / `@PreAuthorize` 등 AOP-based annotation 도 self-invocation 시 무력.

## 해결

### 즉시 (workaround) — `AopContext.currentProxy()`

```java
@Configuration
@EnableAspectJAutoProxy(exposeProxy = true)  // ← 핵심
class AopConfig {}

@Service
public class OrderService {
    @Transactional
    public void process(Order o) {
        save(o);
        ((OrderService) AopContext.currentProxy()).logHistory(o);  // proxy 경유
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logHistory(Order o) { ... }
}
```

작동은 하지만 결합도 + 가독성 나쁨. 임시 해결로만.

### 근본 (3가지)

**(1) 별도 bean 으로 분리** (가장 권장)

```java
@Service
public class OrderService {
    private final OrderHistoryService historyService;

    @Transactional
    public void process(Order o) {
        save(o);
        historyService.logHistory(o);  // ← 다른 bean → proxy 경유
        if (failBusiness()) throw new BizException();
    }
}

@Service
public class OrderHistoryService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logHistory(Order o) { historyRepo.save(...); }
}
```

원칙: **"새 트랜잭션 = 새 책임"** → 책임 분리가 자연스럽게 따라옴.

**(2) `TransactionTemplate` programmatic** — 같은 클래스 안에서 분리 트랜잭션 필요할 때

```java
@Service
public class OrderService {
    private final TransactionTemplate newTxTemplate;

    public OrderService(PlatformTransactionManager tm) {
        this.newTxTemplate = new TransactionTemplate(tm);
        this.newTxTemplate.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
    }

    @Transactional
    public void process(Order o) {
        save(o);
        newTxTemplate.executeWithoutResult(status -> historyRepo.save(...));
        if (failBusiness()) throw new BizException();
    }
}
```

**(3) ApplicationEvent + `@TransactionalEventListener`** — commit 후 별도 트랜잭션

```java
@Transactional
public void process(Order o) {
    save(o);
    eventPublisher.publishEvent(new OrderProcessedEvent(o));
}

@Component
public class OrderHistoryListener {
    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void onOrderProcessed(OrderProcessedEvent e) {
        historyRepo.save(...);
    }
}
```

장점: process 가 history 를 직접 모름 → 결합도 ↓.

## REQUIRES_NEW 의 위험

새 트랜잭션 = 새 connection. 부모 트랜잭션이 connection 1개 보유 중 → REQUIRES_NEW 가 추가 1개 요청 → pool 가용량 절반 차지. cascade 깊으면 **dead lock on connection pool** (모든 thread 가 부모 connection 보유 + 자식 connection 대기).

```
Thread A: 부모 conn=1, 자식 conn 대기 (pool 0)
Thread B: 부모 conn=2, 자식 conn 대기 (pool 0)
...
pool size N → N/2 동시 처리만 가능
```

→ REQUIRES_NEW 가 정말 필요한지 검토. 가능하면 `AFTER_COMMIT` event 패턴.

## 재발 방지

- **PR 검토 룰**: `@Transactional(propagation = REQUIRES_NEW)` 추가 시 (a) 같은 bean 호출 아닌지 (b) connection pool size 영향 검토
- **테스트**: 부모 트랜잭션 rollback → 자식 commit 검증
  ```java
  @Test
  @Transactional  // outer (test) tx 가 모든 것 rollback
  void historyShouldCommitEvenIfBusinessFails() {
      assertThrows(BizException.class, () -> service.process(order));
      // 같은 트랜잭션이라 history 도 보이지 않음 — 별도 read query 필요
      var history = TransactionTemplate.execute(new SimpleTransactionStatus(),
          () -> historyRepo.findByOrderId(order.getId()));
      assertThat(history).isPresent();
  }
  ```
- **ArchUnit**: self-invocation 차단 — 같은 클래스 안 `@Transactional` 메서드 직접 호출 금지

## 관련

- [[backend]]
- [[aop-self-invocation]] — 같은 원인의 다른 증상 (@Async / @Cacheable)
- [[hikari-connection-leak]] — REQUIRES_NEW 가 connection 압박
- [[../../20-areas/backend/spring/spring|20-areas/spring]]
