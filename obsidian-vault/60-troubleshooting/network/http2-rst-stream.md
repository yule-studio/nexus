---
title: "troubleshooting · HTTP/2 RST_STREAM"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [network, http2, nginx, alb]
home_hub: "60-troubleshooting"
related: ["[[network]]"]
---

# HTTP/2 — RST_STREAM 연속 발생

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | network | 최초 |

## 증상

```
RST_STREAM with error code REFUSED_STREAM
HTTP_REQUEST_TIMEOUT (NO_ERROR)
```

```bash
curl --http2 -v https://api.example.com/x
# < HTTP/2 stream 1 was reset (error code 8)
```

## 원인

1. Server max_concurrent_streams 한도 초과
2. CVE-2023-44487 (Rapid Reset attack) 방어 — 짧은 시간 RST 다수 시 connection close
3. Client cancel (사용자 page leave)
4. Server side abort (timeout / shutdown)

## 해결

### Server max_concurrent_streams 조정

**Nginx**:
```nginx
http2_max_concurrent_streams 128;     # default 128, 늘려도 OK
http2_recv_buffer_size 256k;
http2_max_field_size 16k;
http2_max_header_size 32k;
```

**Spring Boot (Tomcat embed)**:
```yaml
server:
  http2.enabled: true
  tomcat:
    max-http-form-post-size: 2MB
    max-connections: 8192
```

**ALB**:
HTTP/2 default enable. 자동 max streams.

### CVE-2023-44487 — Rapid Reset 방어

```nginx
# 새 stream count rate limit
http2_max_concurrent_streams 100;
```

```yaml
# Spring Boot — embedded Tomcat 9.0.83+ / 10.1.16+
# 자동 fix 포함 (upgrade)
```

→ patched version 사용 (Tomcat / Netty / Jetty / nginx 모두 fix).

### Client cancel — 정상 동작

브라우저가 page leave / abort → RST_STREAM `CANCEL` (error 8). Server log 에 나오지만 정상. monitoring 에서 제외.

```
HTTP/2 RST_STREAM error codes:
  0 NO_ERROR              # 정상 종료
  1 PROTOCOL_ERROR        # spec 위반
  2 INTERNAL_ERROR        # server bug
  3 FLOW_CONTROL_ERROR    # window 초과
  4 SETTINGS_TIMEOUT
  5 STREAM_CLOSED         # 이미 닫힌 stream
  6 FRAME_SIZE_ERROR
  7 REFUSED_STREAM        # ← server 가 거절
  8 CANCEL                # ← client / server 가 cancel (정상)
  9 COMPRESSION_ERROR
  10 CONNECT_ERROR
  11 ENHANCE_YOUR_CALM    # rate limit
  12 INADEQUATE_SECURITY  # TLS 약함
```

### gRPC over HTTP/2

```go
// Server
s := grpc.NewServer(
    grpc.MaxConcurrentStreams(1000),
    grpc.KeepaliveParams(keepalive.ServerParameters{
        MaxConnectionIdle: 5 * time.Minute,
        Time:              30 * time.Second,
        Timeout:           5 * time.Second,
    }),
)
```

```go
// Client
conn, _ := grpc.Dial(addr,
    grpc.WithKeepaliveParams(keepalive.ClientParameters{
        Time:                30 * time.Second,
        Timeout:             5 * time.Second,
        PermitWithoutStream: true,
    }),
)
```

## 진단

```bash
# nghttp — HTTP/2 디버그
nghttp -v https://api.example.com/x

# nginx error log
tail -f /var/log/nginx/error.log | grep -i http2

# Wireshark capture — RST_STREAM frame
tcpdump -i any -w h2.pcap port 443
# Wireshark: filter "http2.flags.end_stream"
```

## 재발 방지

- nginx / Tomcat / Jetty / Netty 모두 CVE-2023-44487 patch 버전
- max_concurrent_streams 부하 테스트로 검증
- RST_STREAM CANCEL (8) 은 log noise — filter
- 모니터링: RST_STREAM rate (excluding code 8)

## 관련

- [[network]] / [[tcp-keepalive]] / [[../backend/spring-security-401]]
