---
title: "60-troubleshooting/network — Network 트러블 카탈로그"
kind: knowledge
status: current
created_at: 2026-05-27
tags: [troubleshooting, network, tls, dns, cors, http, hub]
home_hub: "60-troubleshooting"
---

# Network 트러블 카탈로그

> TLS / DNS / CORS / proxy / firewall / sandbox 에서 자주 만나는 대표 트러블.

**[[../troubleshooting|↑ 60-troubleshooting]]**

---

## 카탈로그

| #  | 증상 / 에러                                                                  | 한 줄 원인                              | 핵심 해결                                                       | 깊이 노트                                |
| -- | ------------------------------------------------------------------------ | ---------------------------------- | ----------------------------------------------------------- | ------------------------------------ |
| 1  | **TLS — unable to verify first certificate**                             | intermediate cert 누락 / chain 불완전   | fullchain.pem 제공 / ACM / cert-manager                       | [[tls-handshake-fail]]               |
| 2  | **TLS — certificate signed by unknown authority**                        | trust store 에 root CA 없음           | update-ca-certificates / corporate CA bundle               | [[tls-unknown-authority]]            |
| 3  | **TLS SNI mismatch**                                                     | nginx server_name / SAN 불일치        | server_name 정확 / SAN 에 hostname / default_server          | [[tls-sni-mismatch]]                 |
| 4  | **CORS preflight 차단**                                                    | server OPTIONS 안 받음 / 헤더 누락      | Spring/Express CORS middleware / OPTIONS permitAll          | [[../frontend/cors-preflight-blocked]] |
| 5  | **DNS resolve fail / stale cache**                                       | DNS down / TTL 길음 / JVM cache      | dig +trace / flush / TTL 60-300s / JVM ttl                  | [[dns-resolve-fail]]                 |
| 6  | **Corporate proxy 차단**                                                   | corporate firewall                 | HTTP_PROXY / TLS interception 대응 / corporate CA           | [[corporate-proxy]]                  |
| 7  | **IPv6 vs IPv4 connect timeout**                                         | host AAAA 만 / client IPv4 only     | `--ipv4` / `--dns-result-order=ipv4first`                  | [[ipv6-vs-ipv4]]                     |
| 8  | **TCP keepalive timeout — connection 끊김**                                | NAT / LB idle timeout              | application keepalive ping / LB idle_timeout / tcp_keepalive | [[tcp-keepalive]]                    |
| 9  | **WebSocket subprotocol / proxy upgrade**                                | client/server subprotocol / Upgrade header | server subprotocol / nginx Upgrade+Connection / ALB        | [[websocket-subprotocol-mismatch]]   |
| 10 | **HTTP/2 RST_STREAM**                                                    | max_concurrent_streams / cancel    | max_streams 조정 / CVE-2023-44487 patch / error code 8 정상 | [[http2-rst-stream]]                 |
| 11 | **Connection refused**                                                   | service down / port / SG block     | listen 확인 / bind 0.0.0.0 / SG inbound / K8s endpoints     | [[connection-refused]]               |
| 12 | **AWS VPC — task 가 public internet 못 감**                                  | NAT / VPC Endpoint 없음            | NAT Gateway / VPC Endpoint (ECR/S3/Secrets/Logs)            | [[vpc-public-internet]]              |
| 13 | **iptables / Docker traffic drop**                                       | DOCKER chain / FORWARD DROP        | DOCKER-USER chain / ufw + after.rules / backup/restore     | [[iptables-docker-block]]            |
| 14 | **Claude Code sandbox block**                                            | auto-mode classifier 가 외부 영향 의심   | scope 분리 / explicit prompt / `--dangerously-skip-permissions` | [[claude-sandbox-block]]             |
| 15 | **DNS 가 옛 IP 캐시 — deploy 후**                                              | TTL 길게 / JVM 영구 cache            | TTL 60-300s / LB-based routing / JVM ttl=60                | [[dns-resolve-fail]] (통합)            |

## 자주 쓰는 진단 명령

```bash
# DNS
dig +trace example.com
nslookup example.com 8.8.8.8
host -t MX example.com

# TLS
openssl s_client -connect host:443 -servername host -showcerts
curl -vk https://host/path
ssllabs.com (외부 진단)

# Connectivity
nc -zv host port
mtr host
traceroute -T -p 443 host
ss -tnp / netstat -an

# HTTP
curl -v / -I / --resolve host:443:1.2.3.4
nghttp -v https://host
```

## 관련

- [[../troubleshooting]]
- [[../devops/devops|devops 트러블]] — VPC / SG / NAT / LB 경계
- [[../frontend/frontend|frontend 트러블]] — CORS / Service Worker 경계
- [[../ai-agent/ai-agent|ai-agent 트러블]] — sandbox / classifier block
