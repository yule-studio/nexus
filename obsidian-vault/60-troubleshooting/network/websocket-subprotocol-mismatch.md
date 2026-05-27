---
title: "troubleshooting · WebSocket Sec-WebSocket-Protocol mismatch"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [network, websocket, http, nginx, proxy]
home_hub: "60-troubleshooting"
related: ["[[network]]", "[[tcp-keepalive]]"]
---

# WebSocket 연결 실패 — subprotocol / proxy upgrade

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | network | 최초 |

## 증상

```
WebSocket connection to 'wss://example.com/ws' failed:
Error during WebSocket handshake: Sent non-empty 'Sec-WebSocket-Protocol' header but no response was received

또는
HTTP 200 OK (instead of 101 Switching Protocols)
```

## 원인

1. Server 가 client 의 subprotocol 거절
2. Proxy (nginx / ALB) 가 Upgrade header 전달 안 함
3. CSP / CORS 가 wss:// 차단

## 해결

### Server — subprotocol 명시

```java
// Spring WebSocket
@Configuration
@EnableWebSocket
public class WsConfig implements WebSocketConfigurer {
    public void registerWebSocketHandlers(WebSocketHandlerRegistry r) {
        r.addHandler(new MyHandler(), "/ws")
         .setAllowedOriginPatterns("*")
         .addInterceptors(new HttpSessionHandshakeInterceptor());
    }
}

public class MyHandler extends TextWebSocketHandler {
    // SubProtocolCapable 구현
}
```

```js
// Client 가 subprotocol 명시
const ws = new WebSocket(url, ['v1.json', 'v2.protobuf']);
// Server 가 둘 중 하나 응답 또는 빈 응답
ws.onopen = () => console.log('selected:', ws.protocol);
```

### Nginx — Upgrade header proxy

```nginx
location /ws {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;        # ← 핵심
    proxy_set_header Connection "upgrade";          # ← 핵심
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_read_timeout 86400;       # 24시간 — long-lived
    proxy_send_timeout 86400;
}
```

`Connection: upgrade` 가 default 없음 → 명시 필수.

### ALB
```hcl
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  certificate_arn   = aws_acm_certificate.main.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

resource "aws_lb" "main" {
  idle_timeout = 3600   # WebSocket — 길게
}
```

ALB 는 WebSocket native 지원. idle_timeout 만 길게.

### CSP — wss 허용

```
Content-Security-Policy: connect-src 'self' wss://example.com
```

### Sec-WebSocket-Protocol 동작

```
Client → Server:
  Sec-WebSocket-Protocol: v1.json, v2.protobuf

Server → Client (option 1):
  Sec-WebSocket-Protocol: v1.json    # 선택
Server → Client (option 2):
  (header 없음)                       # 어느 것도 지원 안 함 → client 가 close 결정
```

→ client 가 명시했는데 server 가 응답 안 하면 일부 client (Chrome) 가 reject.

## STOMP over WebSocket

```js
const stompClient = new Client({
    brokerURL: 'wss://example.com/ws',
    connectHeaders: { login: 'user', passcode: 'pass' },
    reconnectDelay: 5000,
    heartbeatIncoming: 10000,
    heartbeatOutgoing: 10000,
});
```

```java
// Spring STOMP
@MessageMapping("/hello")
public void hello(@Payload HelloMessage msg) { ... }
```

## 진단

```bash
# WebSocket handshake 확인
curl -i -N \
  -H "Connection: Upgrade" \
  -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" \
  -H "Sec-WebSocket-Key: $(head -c 16 /dev/urandom | base64)" \
  -H "Sec-WebSocket-Protocol: v1.json" \
  https://example.com/ws

# 응답 status 가 101 이면 성공
```

```
DevTools Network tab → WS filter
- Headers / Frames / Messages
```

## 재발 방지

- proxy config 에 Upgrade / Connection header 표준 (nginx-snippet)
- subprotocol 변경 시 client / server 동시 deploy
- LB idle timeout vs application heartbeat 일치
- 모니터링: WebSocket connection count / reconnect rate

## 관련

- [[network]] / [[tcp-keepalive]]
