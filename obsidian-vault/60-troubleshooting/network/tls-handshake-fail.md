---
title: "troubleshooting · TLS handshake — unable to verify first certificate"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [network, tls, ssl, certificate, chain]
home_hub: "60-troubleshooting"
related: ["[[network]]", "[[../devops/tls-cert-renewal]]"]
---

# TLS — `unable to verify the first certificate` (intermediate 누락)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | network | 최초 |

## 증상

```
SSL routines:OPENSSL_internal:CERTIFICATE_VERIFY_FAILED
Error: unable to verify the first certificate

(node:1234) Error: unable to verify the first certificate
    at TLSSocket.onConnectSecure (_tls_wrap.js:1502:34)
```

- 브라우저는 정상 (자체 truststore 가 missing intermediate 보완)
- curl / Node / Python client 가 실패
- SSL Labs 가 "Chain issues: Incomplete" 경고

## 원인

서버가 client 에 보낸 cert chain 이 incomplete. server cert → intermediate → root 의 체인이 끊어짐. client 의 truststore 에 root 만 있을 때 intermediate 가 빠지면 chain 못 구성.

## 해결

### Server 측 — full chain 제공
```nginx
ssl_certificate /etc/ssl/fullchain.pem;   # server cert + intermediate(s)
ssl_certificate_key /etc/ssl/privkey.pem;
```

Let's Encrypt 의 `fullchain.pem` 가 정확히 이 형식 (cert + intermediate).

Apache:
```
SSLCertificateFile      /etc/ssl/server.crt
SSLCertificateChainFile /etc/ssl/intermediate.crt
```

또는 Apache 2.4.8+ 는 SSLCertificateFile 에 chain 같이.

### Verify
```bash
openssl s_client -connect example.com:443 -servername example.com -showcerts
# 출력에 server cert + intermediate cert 모두 보여야 함
# "Verify return code: 0 (ok)" 확인
```

```bash
# SSL Labs (외부 진단)
# https://www.ssllabs.com/ssltest/analyze.html?d=example.com
# → "Chain issues" 섹션 보기
```

### Client 측 — CA bundle 직접 지정 (정 안 되면)
```bash
# curl
curl --cacert /etc/ssl/certs/ca-certificates.crt https://example.com

# Node.js
NODE_EXTRA_CA_CERTS=/path/to/ca-bundle.crt node app.js

# Python (requests)
requests.get(url, verify='/path/to/ca-bundle.crt')

# Java
keytool -import -alias myroot -file root.crt -keystore $JAVA_HOME/lib/security/cacerts
```

### Self-signed cert (internal / dev)
```bash
# Dev 환경만 — 검증 끄기 (절대 prod X)
curl -k https://internal.example.com   # 검증 skip
NODE_TLS_REJECT_UNAUTHORIZED=0
```

→ MITM 위험. internal CA 를 client truststore 에 추가하는 게 옳음.

## ACM / cert-manager 사용 시 자동

- AWS ACM 가 발급한 cert 는 자동으로 full chain
- cert-manager (K8s) 도 자동
- Let's Encrypt certbot 의 `fullchain.pem` 사용

→ wildcards / 자체 CA 만 chain 수동 관리.

## 진단

```bash
# 1. Full chain 검증
openssl s_client -connect example.com:443 -servername example.com 2>&1 | \
  grep -E '(Certificate chain|Verify return)'

# 2. Cert 의 issuer
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | \
  openssl x509 -noout -issuer

# 3. Intermediate 가 도달 가능한지
openssl s_client -connect example.com:443 -servername example.com -showcerts < /dev/null 2>/dev/null | \
  awk '/BEGIN CERTIFICATE/,/END CERTIFICATE/' | \
  awk -v RS= 'NR==2 {print > "intermediate.pem"}'
openssl verify intermediate.pem

# 4. SSL Labs API
curl "https://api.ssllabs.com/api/v3/analyze?host=example.com"
```

## 재발 방지

- ALB / ACM 사용 — chain 자동
- cert-manager (K8s) 표준
- 수동 nginx config 변경 시 fullchain.pem 검증 의무
- 모니터링: SSL Labs API 주기 검사 (A+ grade 유지)

## 관련

- [[network]] / [[tls-unknown-authority]] / [[tls-sni-mismatch]] / [[../devops/tls-cert-renewal]]
