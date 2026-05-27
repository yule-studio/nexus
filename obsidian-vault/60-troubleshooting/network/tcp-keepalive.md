---
title: "troubleshooting · TCP keepalive — long-lived connection 끊김"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [network, tcp, keepalive, nat, lb]
home_hub: "60-troubleshooting"
related: ["[[network]]"]
---

# TCP keepalive timeout — connection 끊김

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | network | 최초 |

## 증상

- WebSocket / DB connection / long-polling 가 N 분 후 끊김
- application 측은 정상 — `read tcp ...: connection reset by peer`
- traffic 없을 때만 발생, 활발한 connection 는 OK

## 원인

NAT / LB / firewall 의 idle timeout. ALB default 60s, NAT 보통 5-10분. idle connection 을 silently drop. application 이 다음에 send 시 RST.

```
client ---- 5분 idle ---- NAT silently drop
client → send → NAT 에 entry 없음 → drop / RST
```

## 해결

### Application keepalive — idle 보다 짧게

**Hikari (JDBC)**:
```yaml
spring.datasource.hikari:
  max-lifetime: 1800000   # 30분 — DB wait_timeout 보다 짧게
  idle-timeout: 600000    # 10분
  keepalive-time: 30000   # 30초 마다 ping (Hikari 4.0+)
```

**WebSocket**:
```js
// Client
const ws = new WebSocket(url);
const pingInterval = setInterval(() => {
  if (ws.readyState === WebSocket.OPEN) ws.send(JSON.stringify({ type: 'ping' }));
}, 30000);   // 30초 마다

// Server (Spring)
@Bean
public WebSocketHandler handler() {
    return new TextWebSocketHandler() {
        @Override
        public void afterConnectionEstablished(WebSocketSession session) {
            // pong 받으면 timer reset
        }
    };
}
```

```yaml
# Spring WebSocket — ping
spring:
  websocket:
    server:
      max-binary-message-size: 65536
```

**HTTP keep-alive header**:
```
Keep-Alive: timeout=120, max=1000
```

### OS level TCP keepalive

```bash
# Linux — 시스템 전역
sudo sysctl -w net.ipv4.tcp_keepalive_time=60       # 60초 후 첫 probe
sudo sysctl -w net.ipv4.tcp_keepalive_intvl=10      # 10초 간격
sudo sysctl -w net.ipv4.tcp_keepalive_probes=3      # 3회 실패 후 close
```

```java
// Socket-level
Socket socket = new Socket();
socket.setKeepAlive(true);   // OS default timing 사용
```

```go
conn, _ := net.Dial("tcp", addr)
tcp := conn.(*net.TCPConn)
tcp.SetKeepAlive(true)
tcp.SetKeepAlivePeriod(30 * time.Second)
```

### ALB / NLB idle timeout 조정

```hcl
resource "aws_lb" "main" {
  name               = "myapp-lb"
  load_balancer_type = "application"
  idle_timeout       = 300   # 5분 (default 60s)
}
```

→ long-lived API 는 300+. WebSocket 은 NLB (4000s default) 또는 ALB+keepalive.

### Database — server side wait_timeout
```sql
-- MySQL
SHOW VARIABLES LIKE 'wait_timeout';   -- default 28800 (8h)
-- 또는 client 측 connection lifetime 조정

-- Postgres
SHOW idle_in_transaction_session_timeout;
```

## 진단

```bash
# Connection 의 상태
ss -tnp state established | grep <pid>
netstat -an | grep ESTABLISHED | wc -l

# 끊긴 시점 추적 — tcpdump
sudo tcpdump -i any -w trace.pcap host api.example.com
# Wireshark 로 분석 — RST flag

# socket option 확인
ss -tnpo | grep keepalive

# 시스템 keepalive 설정
sysctl net.ipv4.tcp_keepalive_time net.ipv4.tcp_keepalive_intvl net.ipv4.tcp_keepalive_probes
```

## 재발 방지

- application-level keepalive (WebSocket ping, JDBC `keepalive-time`)
- LB idle timeout 가 application keepalive 보다 길게
- OS tcp_keepalive_time 짧게 (60-120s) for long-lived service
- 모니터링: connection reset / error rate

## 관련

- [[network]] / [[../backend/hikari-connection-leak]] / [[websocket-subprotocol-mismatch]]
