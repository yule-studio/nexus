---
title: "troubleshooting · TLS certificate signed by unknown authority"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [network, tls, ssl, ca, truststore]
home_hub: "60-troubleshooting"
related: ["[[network]]", "[[tls-handshake-fail]]"]
---

# TLS — `certificate signed by unknown authority`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | network | 최초 |

## 증상

```
x509: certificate signed by unknown authority
SSL_ERROR_UNKNOWN_CA_ALERT
PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException
```

- corporate proxy (Zscaler / Palo Alto) 가 TLS intercept
- internal CA 사용한 internal service
- Docker container 의 truststore 가 outdated

## 원인

Client 의 trust store 에 cert 의 root CA 없음.

## 해결

### Linux — system truststore 에 CA 추가
```bash
# Debian/Ubuntu
sudo cp my-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates

# RHEL/Fedora
sudo cp my-ca.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust

# Alpine
cp my-ca.crt /usr/local/share/ca-certificates/
update-ca-certificates
```

### Docker — base image 에 CA 빌드
```dockerfile
FROM ubuntu:22.04
COPY corporate-ca.crt /usr/local/share/ca-certificates/
RUN update-ca-certificates
```

### Node.js
```bash
NODE_EXTRA_CA_CERTS=/path/to/corporate-ca.crt node app.js
```

```js
// 또는 require 시점에
process.env.NODE_EXTRA_CA_CERTS = '/path/to/ca.crt';
```

### Python
```bash
# requests
export REQUESTS_CA_BUNDLE=/path/to/ca-bundle.crt
export SSL_CERT_FILE=/path/to/ca-bundle.crt

# pip
pip config set global.cert /path/to/ca-bundle.crt
```

### Java
```bash
keytool -import -trustcacerts -alias corporate -file ca.crt \
  -keystore $JAVA_HOME/lib/security/cacerts -storepass changeit -noprompt

# Spring Boot
java -Djavax.net.ssl.trustStore=/path/to/cacerts \
     -Djavax.net.ssl.trustStorePassword=changeit \
     -jar app.jar
```

### Go
```go
// 시스템 truststore 사용 (default)
// 추가 CA 만 명시
certPool, _ := x509.SystemCertPool()
ca, _ := os.ReadFile("ca.crt")
certPool.AppendCertsFromPEM(ca)
client := &http.Client{
    Transport: &http.Transport{
        TLSClientConfig: &tls.Config{RootCAs: certPool},
    },
}
```

### Curl
```bash
# CA 명시
curl --cacert /path/to/ca.crt https://internal.example.com

# 또는 system bundle
curl --capath /etc/ssl/certs https://...

# Skip (dev only — 절대 prod X)
curl -k https://...
```

## Corporate proxy (Zscaler / Palo Alto / mitmproxy)

회사 network 가 TLS inspection 하면 모든 외부 https 가 corporate CA 로 re-sign:

```bash
# 회사 IT 에서 받은 CA crt 를 모든 client 에 설치
# - OS truststore (위 명령)
# - Docker image build 에 포함
# - Node / Python / Java 환경 변수
```

dev container / CI runner 도 마찬가지.

## 진단

```bash
# Cert chain
openssl s_client -connect example.com:443 -showcerts

# Trust 확인
openssl verify -CAfile /etc/ssl/certs/ca-certificates.crt server.crt

# Python
python -c "import ssl; print(ssl.get_default_verify_paths())"

# Node
node -e "console.log(process.env.NODE_EXTRA_CA_CERTS)"
node -e "require('tls').rootCertificates.forEach((c, i) => console.log(i, c.slice(28, 70)))"

# Java
keytool -list -keystore $JAVA_HOME/lib/security/cacerts -storepass changeit | grep -i corporate
```

## 재발 방지

- Corporate CA 는 base Docker image / CI runner image 에 baked in
- Setup script (`bootstrap.sh`) 가 dev machine 에 자동 설치
- `update-ca-certificates` 가 deploy script 에 포함

## 관련

- [[network]] / [[tls-handshake-fail]] / [[corporate-proxy]] / [[../devops/tls-cert-renewal]]
