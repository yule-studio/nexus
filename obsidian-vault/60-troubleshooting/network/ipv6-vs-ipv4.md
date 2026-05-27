---
title: "troubleshooting · IPv6 vs IPv4 connect timeout"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [network, ipv6, ipv4, dns, dual-stack]
home_hub: "60-troubleshooting"
related: ["[[network]]", "[[dns-resolve-fail]]"]
---

# IPv6 connect timeout — host 는 AAAA 만 응답

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | network | 최초 |

## 증상

- macOS / Linux 에서 `curl https://...` 30초 hang 후 timeout
- 같은 host 가 다른 client / 브라우저에서는 정상
- IPv4 only network (corporate / cellular) 에서 발생

## 원인

호스트가 AAAA (IPv6) 와 A (IPv4) 둘 다 응답. happy eyeballs 알고리즘이 IPv6 우선 시도 → IPv4 only network 에서는 timeout 후 IPv4 fallback. 또는 IPv6 만 응답 → 영원히 timeout.

```bash
dig api.example.com A
api.example.com.  60  IN  A    1.2.3.4

dig api.example.com AAAA
api.example.com.  60  IN  AAAA 2001:db8::1
```

## 해결

### Client 측 — IPv4 강제

**curl**:
```bash
curl -4 https://api.example.com
# 또는
curl --ipv4 https://...
```

**Node.js**:
```bash
node --dns-result-order=ipv4first app.js
```

```js
// 또는 코드
const dns = require('dns');
dns.setDefaultResultOrder('ipv4first');
```

**Python**:
```python
import socket
# IPv4 강제 — global hack
socket.getaddrinfo = lambda *args, **kwargs: [(socket.AF_INET, *rest) for *rest, _, _, _ in socket.getaddrinfo(*args, **kwargs) if rest[0] == socket.AF_INET]
```

**Java**:
```bash
java -Djava.net.preferIPv4Stack=true -jar app.jar
```

### Server 측 — IPv4 유지

DNS 에 AAAA 추가 안 함. 단 IPv6 가 표준이 되어가므로 (mobile / ISP), prod 는 dual stack 권장.

### /etc/hosts 우회 (긴급)
```
1.2.3.4    api.example.com   # IPv4 강제
```

### Network 진단

```bash
# DNS 양쪽 확인
dig api.example.com A
dig api.example.com AAAA

# IPv6 connectivity (host 에서)
ping6 google.com
curl -6 https://google.com

# 둘 다 가능한지
curl -v https://example.com 2>&1 | grep "Trying"
# Trying 2001:db8::1...
# Trying 1.2.3.4...
```

## Docker 함정

```yaml
# docker-compose.yml
services:
  app:
    networks:
      default:
        ipv6_address: ...   # 의도된 IPv6
```

또는 Docker daemon `ipv6: true` 설정. 의도 안 한 IPv6 enable 가 dual-stack 만들어 위 문제 발생.

## K8s — dual stack
```yaml
# Service
spec:
  ipFamilies: [IPv4, IPv6]   # 또는 [IPv4] 만
  ipFamilyPolicy: PreferDualStack
```

## 진단

```bash
# 어느 protocol 으로 connect 시도하는지
curl -v --connect-timeout 5 https://api.example.com

# 둘 다 try
strace -f -e trace=network curl https://api.example.com 2>&1 | grep connect

# DNS resolver order
cat /etc/gai.conf | grep precedence
```

## 재발 방지

- Client 의 default 가 dual-stack — `--dns-result-order=ipv4first` 같은 옵션 명시
- Server 측은 dual-stack 의도적 enable / disable (실수 X)
- 모니터링: client error log 의 connection timeout 패턴 추적

## 관련

- [[network]] / [[dns-resolve-fail]] / [[connection-refused]]
