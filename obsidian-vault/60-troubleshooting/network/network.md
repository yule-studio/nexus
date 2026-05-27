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

| #  | 증상 / 에러                                                                  | 한 줄 원인                              | 핵심 해결                                                                  |
| -- | ------------------------------------------------------------------------ | ---------------------------------- | ---------------------------------------------------------------------- |
| 1  | **TLS handshake fail — `unable to verify the first certificate`**        | intermediate cert 누락 / chain 불완전   | 서버가 full chain 제공 (fullchain.pem) / `openssl s_client -showcerts`     |
| 2  | **TLS — `certificate signed by unknown authority`**                      | client trust store 에 root CA 없음    | root CA 추가 / corporate CA bundle (`SSL_CERT_FILE`) / `update-ca-certificates` |
| 3  | **TLS SNI mismatch — `Hostname/IP does not match certificate's altnames`** | TLS SNI 미설정 / 다른 hostname 의 cert 받음 | curl `--resolve` / nginx `server_name` / SAN 에 도메인 등록                  |
| 4  | **CORS preflight 차단**                                                    | 서버가 OPTIONS 안 받거나 헤더 누락             | server CORS middleware: `Access-Control-Allow-Origin/Methods/Headers` + credentials 옵션 |
| 5  | **DNS resolve fail — `getaddrinfo ENOTFOUND`**                           | local DNS / TTL / DNS 서버 down      | `dig +trace <host>` / `nslookup` / `/etc/resolv.conf` / `dscacheutil -flushcache` (macOS) |
| 6  | **DNS 가 옛 IP 캐시 — deploy 후 옛 서버로 traffic**                                | TTL 길게 설정 / negative cache         | TTL 짧게 (60-300s) / blue-green 시 DNS 가 아닌 LB 라우팅                       |
| 7  | **Corporate proxy 차단 — `npm install ECONNREFUSED`**                      | corporate firewall                 | `HTTP_PROXY` / `HTTPS_PROXY` env / `npm config set proxy`              |
| 8  | **IPv6 vs IPv4 — connect timeout on IPv6 only**                          | host 가 AAAA 만 응답 / client IPv4 only | `--ipv4` 강제 / `node --dns-result-order=ipv4first`                     |
| 9  | **TCP keepalive timeout — long-lived connection 끊김**                     | NAT / LB idle timeout              | application keepalive ping (간격 < idle timeout) / `tcp_keepalive_*` sysctl |
| 10 | **WebSocket 연결 실패 — `Sec-WebSocket-Protocol mismatch`**                  | client subprotocol vs server 불일치   | 양쪽 subprotocol 일치 / proxy (nginx) `proxy_set_header Upgrade`           |
| 11 | **HTTP/2 — `RST_STREAM` 연속 발생**                                          | server side abort / client cancel / max_concurrent_streams 초과 | `nghttp -v` 디버그 / `http2_max_concurrent_streams` 조정                  |
| 12 | **`Connection refused`**                                                 | 서비스 down / 잘못된 port / firewall    | `nc -zv host port` / `lsof -i :port` / SG inbound 확인                  |
| 13 | **AWS VPC — task 가 public internet 못 나감**                                | private subnet without NAT / route table 잘못 | NAT Gateway 추가 / VPC Endpoint (Interface — ECR/Secrets/SSM/Logs)     |
| 14 | **iptables / nftables 가 traffic drop**                                   | DOCKER chain 우선순위 / FORWARD policy DROP | `iptables -L -v -n` / Docker 가 자동 관리 (수동 변경 금지)                       |
| 15 | **Claude Code sandbox block — `python -m yule_orchestrator` 거부**         | auto-mode classifier 가 외부 영향 의심한 명령 차단 | scope 분리 / explicit prompt clarification — [[../ai-agent/ai-agent\|ai-agent 트러블]] |

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
