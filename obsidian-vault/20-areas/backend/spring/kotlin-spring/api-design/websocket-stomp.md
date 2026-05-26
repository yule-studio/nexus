---
title: "WebSocket + STOMP — Kotlin Spring Boot"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-26T14:10:00+09:00
tags:
  - backend
  - kotlin-spring
  - api-design
  - recipe
  - websocket
  - stomp
  - realtime
---

# WebSocket + STOMP — Kotlin Spring Boot

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-26 | engineering-agent/tech-lead | java-spring 미러링 + interceptor + 함정 |

**[[api-design|↑ api-design hub]]** · **[[../../java-spring/api-design/websocket-stomp|↗ Java 원본]]**

> 📐 공통: [[../common/response-envelope]] · [[../common/security-config]].

---

## 1. 무엇을 만드는가 / 2. 의존성 / 8. heartbeat / 9. 다중 인스턴스 / 10. 함정

(Java 원본 §1, §2, §8, §9 와 동일)

---

## 3. WebSocket 설정

```kotlin
@Configuration
@EnableWebSocketMessageBroker
class WebSocketConfig(
    private val authInterceptor: StompAuthChannelInterceptor,
) : WebSocketMessageBrokerConfigurer {

    override fun registerStompEndpoints(registry: StompEndpointRegistry) {
        registry.addEndpoint("/api/v1/ws")
            .setAllowedOriginPatterns(
                "https://shop.example.com",
                "https://admin.example.com",
                "http://localhost:3000",
            )
            .withSockJS()
    }

    override fun configureMessageBroker(registry: MessageBrokerRegistry) {
        // 내장 simple broker — 단일 인스턴스 OK
        registry.enableSimpleBroker("/topic", "/queue")
            .setHeartbeatValue(longArrayOf(10_000, 10_000))
            .setTaskScheduler(heartbeatScheduler())

        // 다중 인스턴스 + 외부 broker — RabbitMQ STOMP (주석)
        // registry.enableStompBrokerRelay("/topic", "/queue")
        //     .setRelayHost(rabbitHost).setRelayPort(61613)
        //     .setClientLogin(user).setClientPasscode(pwd)

        registry.setApplicationDestinationPrefixes("/app")
        registry.setUserDestinationPrefix("/user")
    }

    override fun configureClientInboundChannel(registration: ChannelRegistration) {
        registration.interceptors(authInterceptor)
    }

    @Bean
    fun heartbeatScheduler(): TaskScheduler = ThreadPoolTaskScheduler().apply {
        poolSize = 2
        setThreadNamePrefix("ws-heartbeat-")
        initialize()
    }
}
```

> **Kotlin idiom 포인트**:
> - `longArrayOf(10_000, 10_000)` — Java `new long[]{...}`.
> - `ThreadPoolTaskScheduler().apply { poolSize = 2; ... }` — 객체 초기화를 scope function 으로.

---

## 4. STOMP CONNECT 시 JWT 인증

```kotlin
@Component
class StompAuthChannelInterceptor(
    private val jwt: JwtTokenProvider,
    private val users: UserQueryService,
    private val subscriptionGuard: StompSubscriptionGuard,
) : ChannelInterceptor {

    private val log = LoggerFactory.getLogger(javaClass)

    override fun preSend(message: Message<*>, channel: MessageChannel): Message<*> {
        val accessor = StompHeaderAccessor.wrap(message)

        when (accessor.command) {
            StompCommand.CONNECT -> {
                val auth = accessor.getFirstNativeHeader("Authorization")
                    ?: throw MessageDeliveryException("missing token")
                if (!auth.startsWith("Bearer ")) throw MessageDeliveryException("invalid bearer")

                val token = auth.removePrefix("Bearer ")
                if (!jwt.validateToken(token) || !jwt.isAccess(token)) {
                    throw MessageDeliveryException("invalid token")
                }

                val email = jwt.getEmail(token)
                val user = users.findActiveByEmail(email)
                    ?: throw MessageDeliveryException("user not found")

                val authentication = UsernamePasswordAuthenticationToken(
                    user, null,
                    listOf(SimpleGrantedAuthority(user.role.asAuthority())),
                )
                accessor.user = authentication
                log.info("WS CONNECT user={}", user.id)
            }

            StompCommand.SUBSCRIBE -> checkSubscriptionPermission(accessor)
            else -> { /* MESSAGE / UNSUBSCRIBE / DISCONNECT — 별도 처리 X */ }
        }
        return message
    }

    private fun checkSubscriptionPermission(accessor: StompHeaderAccessor) {
        val destination = accessor.destination ?: throw MessageDeliveryException("no destination")
        val principal = accessor.user as? Authentication
            ?: throw MessageDeliveryException("subscription requires auth")
        val user = principal.principal as User

        if (!subscriptionGuard.canSubscribe(user, destination)) {
            throw MessageDeliveryException("subscription forbidden: $destination")
        }
    }
}
```

> **Kotlin idiom 포인트**:
> - `when (accessor.command) { StompCommand.CONNECT -> ... }` — Java if/else chain.
> - `auth.removePrefix("Bearer ")` — stdlib.
> - `as? Authentication ?: throw ...` — safe cast + fallback.

---

## 5. 권한 — destination 별 검증

```kotlin
@Component
class StompSubscriptionGuard(
    private val chatMembers: ChatRoomMembershipService,
) {

    fun canSubscribe(user: User, destination: String): Boolean = when {
        destination.startsWith("/topic/admin/") -> user.role in setOf(Role.ADMIN, Role.MASTER)
        destination.startsWith("/topic/chat/") -> {
            val roomId = destination.removePrefix("/topic/chat/")
            chatMembers.isMember(user.id, roomId)
        }
        destination.startsWith("/user/queue/") -> true     // Spring 이 user 매칭 자동
        else -> false                                       // default deny
    }
}
```

> **Kotlin idiom**: `when { ... }` (subject-less) — Java if/else if chain.

---

## 6. Send / Subscribe 패턴

### 6.1 broadcast (전체 / 채팅방)

```kotlin
@Component
class ChatMessageBroadcaster(
    private val ws: SimpMessagingTemplate,
) {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun on(event: ChatMessageSent) {
        ws.convertAndSend(
            "/topic/chat/${event.roomId}",
            ChatMessageDto(event.messageId, event.senderId, event.content, event.sentAt),
        )
    }
}
```

### 6.2 user 개인 큐

```kotlin
ws.convertAndSendToUser(
    userId.value,                                   // principal.name
    "/queue/notifications",
    NotificationDto(/* ... */),
)
// 클라가 SUBSCRIBE /user/queue/notifications 하면 자기것만 받음
```

### 6.3 Controller — STOMP `@MessageMapping`

```kotlin
@Controller
class ChatStompController(
    private val chatService: ChatService,
) {

    @MessageMapping("/chat/{roomId}/send")
    fun send(
        @DestinationVariable roomId: String,
        @Payload req: SendMessageRequest,
        principal: Principal,
    ) {
        val authentication = principal as Authentication
        val senderId = (authentication.principal as User).id
        chatService.send(ChatRoomId(roomId), senderId, req.content)
        // 도메인 이벤트가 broadcast 처리 (§6.1)
    }
}

data class SendMessageRequest(@field:NotBlank @field:Size(max = 4000) val content: String)
```

---

## 7. SecurityConfig — WebSocket endpoint permit

(Java 원본 §7 와 동일 — Kotlin DSL 의 `authorize("/api/v1/ws/**", permitAll)`)

---

## 11. 함정 모음

(원본 [[../../java-spring/api-design/websocket-stomp#10 함정 모음|§10]] 의 10가지 함정 그대로)

### Kotlin 한정 추가 함정

### 함정 11 — `accessor.user` property assignment
Java `accessor.setUser(authentication)` → Kotlin `accessor.user = authentication`. property syntax.

### 함정 12 — `when (accessor.command)` 의 exhaustive
`StompCommand` 가 enum 이지만 exhaustive 강제 안 됨 (Java enum 의 한계). `else -> { }` 명시 권장.

### 함정 13 — `@MessageMapping` 메서드 final
`kotlin("plugin.spring")` 없으면 final → STOMP routing 안 됨. plugin 적용 강제.

---

## 12. 운영 체크리스트

(Java 원본 §11 그대로)

---

## 13. 관련

- [[../../java-spring/api-design/websocket-stomp|↗ Java 원본]]
- [[../common/security-config]] — STOMP CONNECT 인증
- [[api-design|↑ api-design hub]]
