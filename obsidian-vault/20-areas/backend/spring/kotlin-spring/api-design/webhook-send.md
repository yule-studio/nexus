---
title: "Webhook 송신 (서명 / 재시도 / DLQ) — Kotlin"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T14:00:00+09:00
tags:
  - backend
  - kotlin-spring
  - api-design
  - recipe
  - webhook
---

# Webhook 송신 (서명 / 재시도 / DLQ) — Kotlin

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + when expression backoff + 함정 |

**[[api-design|↑ api-design hub]]** · **[[../../java-spring/api-design/webhook-send|↗ Java 원본]]**

> 📐 공통: [[../common/response-envelope]].

---

## 1. 무엇을 만드는가 / 2. 도메인·DB / 8. 수신자 가이드 / 9. 함정

(Java 원본 §1, §2, §8, §9 와 동일)

---

## 3. Outbox enqueue

```kotlin
@Service
class EnqueueWebhookUseCase(
    private val subs: WebhookSubscriptionRepository,
    private val deliveries: WebhookDeliveryRepository,
    private val ids: IdGenerator,
    private val clock: Clock,
    private val json: ObjectMapper,
) {

    @Transactional(propagation = Propagation.MANDATORY)
    fun enqueue(eventType: String, eventId: String, payload: Any) {
        subs.findActiveByEventType(eventType).forEach { sub ->
            runCatching {
                deliveries.save(
                    WebhookDelivery(
                        id = WebhookDeliveryId(ids.next()),
                        subscriptionId = sub.id,
                        eventId = eventId,
                        eventType = eventType,
                        payload = json.valueToTree(payload),
                        status = DeliveryStatus.PENDING,
                        attempts = 0,
                        nextAttemptAt = Instant.now(clock),
                        lastError = null,
                        lastResponseStatus = null,
                        createdAt = Instant.now(clock),
                        sentAt = null,
                    ),
                )
            }.onFailure { ex ->
                if (ex !is DataIntegrityViolationException) throw ex
                // 같은 event_id 가 이미 enqueue 됨 — 무시 (멱등)
            }
        }
    }
}

@Component
class OrderPaidWebhookListener(
    private val enqueue: EnqueueWebhookUseCase,
) {
    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    fun on(event: OrderPaid) {
        enqueue.enqueue(
            "ORDER_PAID",
            event.orderId.value,
            OrderPaidPayload(event.orderId.value, event.amount.amountMinorUnit, event.occurredAt),
        )
    }
}
```

---

## 4. Worker

```kotlin
@Component
class WebhookDeliveryWorker(
    private val deliveries: WebhookDeliveryRepository,
    private val subs: WebhookSubscriptionRepository,
    private val http: WebhookHttpClient,
    private val json: ObjectMapper,
    private val clock: Clock,
) {
    private val log = LoggerFactory.getLogger(javaClass)

    @Scheduled(fixedDelay = 1000)
    @SchedulerLock(name = "webhookDeliveryWorker", lockAtMostFor = "60s", lockAtLeastFor = "5s")
    fun process() {
        deliveries.findDue(Instant.now(clock), 50).forEach { deliverOne(it.id) }
    }

    @Transactional
    fun deliverOne(deliveryId: WebhookDeliveryId) {
        val delivery = deliveries.findByIdForUpdate(deliveryId) ?: return
        if (delivery.status !in setOf(DeliveryStatus.PENDING, DeliveryStatus.RETRY)) return

        val sub = subs.findById(delivery.subscriptionId)
        if (sub == null || !sub.active) {
            delivery.markDeadLetter("subscription inactive")
            deliveries.save(delivery)
            return
        }

        runCatching {
            val payloadStr = json.writeValueAsString(delivery.payload)
            val signature = hmacSha256Hex(payloadStr, sub.secret)

            val response = http.post(
                url = sub.targetUrl,
                body = payloadStr,
                headers = mapOf(
                    "Content-Type" to "application/json",
                    "X-Webhook-Event" to delivery.eventType,
                    "X-Webhook-Delivery" to delivery.id.value,
                    "X-Webhook-Signature" to "sha256=$signature",
                    "X-Idempotency-Key" to delivery.eventId,
                    "User-Agent" to "Shop-Webhook/1.0",
                ),
            )

            if (response.statusCode in 200..299) {
                delivery.markSent(Instant.now(clock), response.statusCode)
            } else {
                delivery.recordFailure(
                    status = response.statusCode,
                    error = response.body,
                    nextAttempt = computeNextAttempt(delivery.attempts + 1),
                )
            }
        }.onFailure { ex ->
            delivery.recordFailure(
                status = null,
                error = ex.message,
                nextAttempt = computeNextAttempt(delivery.attempts + 1),
            )
        }
        deliveries.save(delivery)
    }

    /** Exponential backoff: 30s → 5m → 30m → 2h → 6h → 12h → 24h → DEAD_LETTER */
    private fun computeNextAttempt(attempt: Int): Instant {
        val seconds = when (attempt) {
            1 -> 30L
            2 -> 300L
            3 -> 1800L
            4 -> 7200L
            5 -> 21600L
            6 -> 43200L
            7 -> 86400L
            else -> -1L
        }
        return if (seconds < 0) Instant.MAX else Instant.now(clock).plusSeconds(seconds)
    }

    private fun hmacSha256Hex(body: String, secret: String): String {
        val mac = Mac.getInstance("HmacSHA256").apply {
            init(SecretKeySpec(secret.toByteArray(Charsets.UTF_8), "HmacSHA256"))
        }
        return HexFormat.of().formatHex(mac.doFinal(body.toByteArray(Charsets.UTF_8)))
    }
}
```

> **Kotlin idiom 포인트**:
> - `delivery.status !in setOf(PENDING, RETRY)` — Java `!= X && != Y` 패턴.
> - `response.statusCode in 200..299` — IntRange.
> - `when (attempt) -> when expression` — Java switch + yield.

---

## 5. HTTP Client (timeout / circuit breaker)

```kotlin
@Component
class WebhookHttpClient {

    private val http: HttpClient = HttpClient.newBuilder()
        .connectTimeout(Duration.ofSeconds(5))
        .build()

    @CircuitBreaker(name = "webhook", fallbackMethod = "fallback")
    fun post(url: String, body: String, headers: Map<String, String>): Response {
        val builder = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .timeout(Duration.ofSeconds(10))
            .POST(HttpRequest.BodyPublishers.ofString(body))
        headers.forEach(builder::header)

        val res = runCatching { http.send(builder.build(), HttpResponse.BodyHandlers.ofString()) }
            .getOrElse { throw RuntimeException(it) }
        return Response(res.statusCode(), res.body() ?: "")
    }

    @Suppress("UNUSED_PARAMETER")
    fun fallback(url: String, body: String, headers: Map<String, String>, t: Throwable): Response =
        Response(0, "circuit open: ${t.message}")

    data class Response(val statusCode: Int, val body: String)
}
```

> **Kotlin 한정**: `@CircuitBreaker` 의 `fallbackMethod = "fallback"` 가 호출되려면 fallback 메서드도 같은 visibility + 마지막 파라미터가 `Throwable`. Kotlin 의 expression function 으로 짧게.

(yaml resilience4j 설정은 Java 원본 §5 와 동일)

---

## 6. Subscription 관리 endpoint / 7. Dead Letter 처리

(Java 원본 §6, §7 와 동일 — Controller 패턴은 다른 레시피 참조)

```kotlin
@RestController
@RequestMapping("/api/v1/webhooks/subscriptions")
@PreAuthorize("hasAnyRole('SELLER', 'ADMIN', 'MASTER')")
class WebhookSubscriptionController(
    private val service: WebhookSubscriptionService,
) {

    @PostMapping
    fun create(
        @Valid @RequestBody req: CreateSubRequest,
        auth: Authentication,
    ): ResponseEntity<CommonResponse<SubscriptionCreated>> {
        val sub = service.create(currentSubscriberId(auth), req)
        return CommonResponse.success(
            ResponseCode.OK,
            SubscriptionCreated(sub.id, sub.secret),
            "구독 생성. secret 은 다시 노출되지 않습니다.",
        ).toResponseEntity()
    }

    @PostMapping("/{id}/test")
    fun testSend(
        @PathVariable id: String,
        auth: Authentication,
    ): ResponseEntity<CommonResponse<Unit>> {
        service.sendTest(id, currentSubscriberId(auth))
        return CommonResponse.success<Unit>(ResponseCode.OK, "테스트 이벤트 발송").toResponseEntity()
    }
}
```

---

## 10. 함정 모음

(원본 [[../../java-spring/api-design/webhook-send#9 함정 모음|§9]] 의 10가지 함정 그대로)

### Kotlin 한정 추가 함정

### 함정 11 — `@CircuitBreaker` fallback 메서드 signature
`fallbackMethod` 는 원본 메서드 시그니처 + 마지막 `Throwable`. Kotlin 의 expression function 으로 짧게 쓸 때 visibility / parameter list 정확히.

### 함정 12 — `HttpRequest.Builder` 의 `forEach { (k, v) -> }` vs reference
`headers.forEach(builder::header)` — Map.Entry 받는 BiConsumer 와 매핑. Kotlin Map 의 forEach 도 동일.

### 함정 13 — `Instant.MAX` 와 `Long.MAX` 변환
`computeNextAttempt` 의 `Instant.MAX` 는 plusSeconds 시 오버플로 위험. **별도 sentinel** (예: `Instant.parse("9999-12-31T...")`) 권장.

---

## 11. 운영 체크리스트

(Java 원본 §10 그대로)

---

## 12. 관련

- [[../../java-spring/api-design/webhook-send|↗ Java 원본]]
- [[distributed-lock]] — ShedLock
- [[../common/security-config]]
- [[api-design|↑ api-design hub]]
