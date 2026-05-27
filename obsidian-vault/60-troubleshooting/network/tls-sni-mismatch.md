---
title: "troubleshooting · TLS SNI mismatch"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [network, tls, sni, hostname]
home_hub: "60-troubleshooting"
related: ["[[network]]", "[[tls-handshake-fail]]"]
---

# TLS — `Hostname/IP does not match certificate's altnames`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | network | 최초 |

## 증상

```
Error: Hostname/IP does not match certificate's altnames:
  Host: api.example.com. is not in the cert's altnames:
  DNS:other-app.example.com, DNS:*.other.com
```

- nginx 가 잘못된 default server 응답
- IP 로 직접 접근 (hostname 검증 실패)
- HTTPS LB 가 wildcard cert 제공인데 sub-sub-domain 요청

## 원인

TLS SNI (Server Name Indication) — client 가 handshake 시 hostname 전달, server 가 해당 hostname 의 cert 응답. SNI 가 안 보내지거나 server 가 잘못 매칭하면 wrong cert 제공.

```
curl https://api.example.com  → SNI=api.example.com
nginx → SNI 매칭 안 됨 → default_server 의 cert 응답 (other-app.example.com)
client → SAN 확인 → mismatch → 차단
```

## 해결

### Server (nginx) — server_name 정확히
```nginx
server {
    listen 443 ssl;
    server_name api.example.com;
    ssl_certificate /etc/ssl/api.crt;
    ssl_certificate_key /etc/ssl/api.key;
}

server {
    listen 443 ssl;
    server_name other-app.example.com;
    ssl_certificate /etc/ssl/other.crt;
    ssl_certificate_key /etc/ssl/other.key;
}

# default_server 명시 — 매칭 안 되는 SNI 의 fallback
server {
    listen 443 ssl default_server;
    server_name _;
    return 444;   # connection close without response
}
```

### Cert 의 SAN 확장
```bash
openssl x509 -in cert.crt -noout -text | grep -A 2 "Subject Alternative Name"
# X509v3 Subject Alternative Name:
#     DNS:api.example.com, DNS:*.api.example.com
```

SAN 에 hostname 포함되어야. CN (Common Name) 만으론 modern 브라우저가 인정 X.

```bash
# CSR 만들 때 SAN 명시
openssl req -new -key key.pem -out req.csr -config <(cat <<EOF
[req]
distinguished_name = dn
req_extensions = v3_req
[dn]
CN = api.example.com
[v3_req]
subjectAltName = DNS:api.example.com,DNS:*.api.example.com
EOF
)
```

### Wildcard 한계

```
*.example.com 는 a.example.com 매칭 OK
*.example.com 는 a.b.example.com 매칭 X (한 단계만)
```

→ 다중 단계 필요 시 `*.example.com` + `*.api.example.com` 같이 multiple SAN.

### Client 측 — SNI 명시

```bash
# curl
curl --resolve api.example.com:443:1.2.3.4 https://api.example.com
# IP 로 접근하면서 SNI/host 는 hostname

# openssl
openssl s_client -connect 1.2.3.4:443 -servername api.example.com
```

### Java
```java
// SNI 자동 전송 (default)
URL url = new URL("https://api.example.com/");
HttpsURLConnection conn = (HttpsURLConnection) url.openConnection();
// 단 IP 로 접근 시 hostname verification fail
```

```java
// IP 로 접근 + hostname 검증 우회 (테스트만)
HttpsURLConnection.setDefaultHostnameVerifier((hostname, session) -> true);   // 위험
```

## ALB / CloudFront 의 wildcard

ALB:
```hcl
resource "aws_acm_certificate" "main" {
  domain_name               = "example.com"
  subject_alternative_names = ["*.example.com", "*.api.example.com"]
}
```

→ ALB listener 에 연결. SNI 자동 처리.

## 진단

```bash
# Cert 의 SAN
openssl s_client -connect host:443 -servername host 2>/dev/null | \
  openssl x509 -noout -text | grep -A 2 "Subject Alternative Name"

# 어느 cert 가 반환되는지
openssl s_client -connect host:443 -servername other.host 2>/dev/null | \
  openssl x509 -noout -subject -issuer

# IP 의 default cert (SNI 안 보냄)
openssl s_client -connect 1.2.3.4:443 -noservername 2>/dev/null | \
  openssl x509 -noout -subject
```

## 재발 방지

- nginx server block 의 `server_name` 정확히 + `default_server` 명시
- cert 의 SAN 에 모든 hostname 포함 (CSR 작성 시점)
- ACM cert 의 alternative names 확인
- e2e: 새 domain 추가 시 SSL Labs 검증

## 관련

- [[network]] / [[tls-handshake-fail]] / [[tls-unknown-authority]]
