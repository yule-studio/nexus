---
title: "troubleshooting · DNS resolve fail / stale cache"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [network, dns, ttl, cache]
home_hub: "60-troubleshooting"
related: ["[[network]]"]
---

# DNS resolve fail / stale cache

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | network | 최초 |

## 증상

```
Error: getaddrinfo ENOTFOUND api.example.com
DNS_PROBE_FINISHED_NXDOMAIN
java.net.UnknownHostException: api.example.com
```

또는 — deploy 후 옛 IP 로 traffic:
```
ALB 가 새 IP → DNS 가 옛 IP cache → 사용자가 옛 server hit
```

## 원인

1. DNS record 누락 / 오타
2. DNS server 도달 못함 (resolver 설정 / network)
3. Cache stale — TTL 길음
4. JVM DNS cache 영구 (default Java 8 이전)

## 해결

### 즉시 — 확인 + flush
```bash
# 시스템 DNS 캐시 flush
# macOS
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder

# Linux (systemd-resolved)
sudo systemd-resolve --flush-caches

# Windows
ipconfig /flushdns

# 브라우저 (Chrome)
chrome://net-internals/#dns → Clear host cache
```

### DNS 조회
```bash
# 기본
nslookup api.example.com
dig api.example.com

# 특정 DNS server 로
dig @8.8.8.8 api.example.com
dig @1.1.1.1 api.example.com

# 모든 record type
dig api.example.com ANY

# trace (root → TLD → authoritative)
dig +trace api.example.com

# Reverse
dig -x 1.2.3.4
```

### Resolver 설정
```bash
# Linux — /etc/resolv.conf
nameserver 8.8.8.8
nameserver 1.1.1.1
options timeout:2 attempts:2

# 또는 systemd-resolved
sudo resolvectl status
sudo resolvectl dns wlp3s0 8.8.8.8
```

### TTL 짧게 — deploy 직전
```
# Route53
- type: A
  ttl: 60    # 보통 3600 (1h) → 60 으로 (deploy 전)
  records:
    - 1.2.3.4
```

→ deploy 전에 TTL 60s 로 변경, deploy 완료 후 다시 길게.

### Blue-Green — DNS 가 아닌 LB 라우팅
```
DNS → fixed LB hostname → LB 가 blue/green target group 전환
```

DNS 변경 없이 LB level 에서 즉시 cutover.

### JVM DNS cache
```
# Java 8+ default — networkaddress.cache.ttl = 30 (security manager 있으면 -1)
# Spring Boot
java -Dnetworkaddress.cache.ttl=60 -jar app.jar
```

```java
// 코드로
java.security.Security.setProperty("networkaddress.cache.ttl", "60");
```

→ 영구 cache 면 deploy 후에도 옛 IP. 60초 정도 권장.

### Container DNS
```yaml
# K8s — Pod-level DNS config
spec:
  dnsPolicy: ClusterFirst
  dnsConfig:
    options:
      - name: ndots
        value: "2"     # default 5 → search domain 폭증 차단
      - name: timeout
        value: "2"
```

## EKS DNS 함정

CoreDNS 가 cluster-internal — 외부 DNS query 시 upstream forward. CoreDNS 의 NodeLocal DNSCache 도입 권장 (latency / DNS 부하).

## 진단

```bash
# 호스트의 DNS 가 작동하는지
host -v api.example.com

# Public DNS 와 비교
dig @8.8.8.8 api.example.com
dig @1.1.1.1 api.example.com

# DNS query latency
dig api.example.com | grep "Query time"

# Container 안
kubectl exec <pod> -- nslookup api.example.com
kubectl exec <pod> -- cat /etc/resolv.conf

# DNS 변경 propagation (DNSChecker)
# https://dnschecker.org/#A/api.example.com
```

## 재발 방지

- Production domain 의 TTL 60-300s (긴급 변경 가능)
- Blue-Green 은 LB-based (DNS-based 아님)
- 모니터링: `probe_dns_lookup_time_seconds` (blackbox exporter)
- runbook: deploy 전 DNS TTL 변경 체크리스트

## 관련

- [[network]] / [[connection-refused]] / [[../devops/ecs-task-failed-to-start]]
