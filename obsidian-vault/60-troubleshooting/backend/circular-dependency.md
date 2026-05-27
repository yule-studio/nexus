---
title: "troubleshooting · Circular dependency (Spring Bean creation exception)"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [backend, spring, di, bean, circular]
home_hub: "60-troubleshooting"
related:
  - "[[backend]]"
  - "[[aop-self-invocation]]"
---

# Circular dependency — `Bean currently in creation`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | backend-engineer | 최초 — constructor injection / @Lazy / 책임 분리 |

## 증상

```
The dependencies of some of the beans in the application context form a cycle:

┌─────┐
|  orderService defined in class path resource [.../OrderService.class]
↑     ↓
|  paymentService defined in class path resource [.../PaymentService.class]
↑     ↓
|  orderService (again)
└─────┘

Action:
Relying upon circular references is discouraged and they are prohibited by default.
Update your application to remove the dependency cycle between beans.
```

- Spring Boot 2.6+ 부터 default 로 circular reference 거부
- 시작 시 `BeanCurrentlyInCreationException`

## 가설

- Setter / field injection 으로 우회? → Spring 가 거절 (좋은 기본값)
- `spring.main.allow-circular-references=true` 로 켜? → 작동하지만 근본 해결 아님
- 실제: 설계가 잘못됐다는 신호

## 원인 (root cause)

Constructor injection 의 dependency graph 는 DAG 여야 함. 순환이면 어느 bean 을 먼저 생성할지 결정 불가:

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final PaymentService paymentService;   // ← B 필요
}

@Service
@RequiredArgsConstructor
public class PaymentService {
    private final OrderService orderService;       // ← A 필요
    // chicken-and-egg
}
```

setter injection 으로 우회하면 시작은 되지만, 호출 시점에 한쪽이 null 이거나 partial init 상태일 위험.

## 해결

### 임시 (workaround) — 운영 hotfix

```yaml
spring:
  main:
    allow-circular-references: true   # ← 비추천, 임시만
```

→ 작동하지만 설계 부채 누적. 6개월 후 누군가 또 만남.

### 근본 (5가지)

**(1) 책임 분리 — 가장 옳음**

순환은 보통 "한 클래스가 너무 많이 안다" 의 신호. 공통 책임을 별도 클래스로:

```java
// 공통 로직 추출
@Service
public class OrderPaymentCoordinator {
    private final OrderRepo orderRepo;
    private final PaymentRepo paymentRepo;

    public void process(Long orderId) {
        // orderService + paymentService 가 양쪽 알 필요 없게
    }
}

@Service
public class OrderService {
    private final OrderRepo orderRepo;   // payment 모름
}

@Service
public class PaymentService {
    private final PaymentRepo paymentRepo;   // order 모름
}
```

**(2) 인터페이스 + Event 패턴** — decoupling

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final ApplicationEventPublisher publisher;

    public void create(...) {
        var order = orderRepo.save(...);
        publisher.publishEvent(new OrderCreatedEvent(order));
    }
}

@Component
public class PaymentEventListener {
    private final PaymentService paymentService;

    @EventListener
    public void onOrderCreated(OrderCreatedEvent e) {
        paymentService.charge(e.orderId());
    }
}
```

→ OrderService 가 PaymentService 를 모름 (event 만). 양방향 의존 제거.

**(3) `@Lazy`** — 한쪽을 지연 init

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    @Lazy
    private final PaymentService paymentService;   // proxy 주입 → 실제 호출 시 init
}
```

작동하나 (1)/(2) 가 가능하면 그게 우선. `@Lazy` 는 init 순서만 해결, 설계 문제는 그대로.

**(4) Provider / ObjectFactory**

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final ObjectProvider<PaymentService> paymentServiceProvider;

    public void process(...) {
        paymentServiceProvider.getObject().charge(...);   // 호출 시점에 lookup
    }
}
```

`@Lazy` 와 비슷한 효과, type-safe.

**(5) Setter injection (마지막 수단)**

```java
@Service
public class OrderService {
    private PaymentService paymentService;

    @Autowired
    public void setPaymentService(PaymentService p) { this.paymentService = p; }
}
```

constructor injection 의 immutable / null-safety 장점을 잃음. **권장 안 함**.

## Self-reference 특수 케이스

```java
@Service
public class CommentService {
    @Lazy private final CommentService self;   // self proxy 주입

    public void replyTree(Long id, ...) {
        Comment c = repo.findById(id);
        c.getReplies().forEach(r -> self.replyTree(r.getId(), ...));  // AOP 경유
    }
}
```

→ self-invocation [[aop-self-invocation]] 해결의 또 다른 패턴.

## 진단

```bash
# Spring 이 알려주는 cycle path 정확히 읽기
# "┌─────┐\n  orderService\n  ↑     ↓\n  paymentService\n  └─────┘"
# → 양방향 chain 의 어느 한 쪽을 끊으면 됨

# Bean dependency 시각화
# /actuator/beans → 의존 그래프 추출
curl localhost:8080/actuator/beans | jq '.contexts.application.beans' | head -50
```

## 재발 방지

- **`spring.main.allow-circular-references=false`** 유지 (Spring Boot 2.6+ default)
- **ArchUnit 룰**: package 간 양방향 dependency 금지
  ```java
  @Test
  void packagesShouldHaveNoCircularDeps() {
      slices().matching("..service.(*)..").should().beFreeOfCycles();
  }
  ```
- **PR 검토**: 새 의존 추가 시 graph 영향 검토 (특히 같은 layer 안에서)
- **Domain-driven design**: aggregate / bounded context 명확히 — 같은 aggregate 안 의존 자연스러움, 다른 aggregate 양방향이면 reconsider

## 관련

- [[backend]]
- [[aop-self-invocation]] — proxy 와 self-reference
- 책: Domain-Driven Design (Eric Evans), Effective Java - Item 1 (정적 팩토리)
