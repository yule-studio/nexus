---
title: "troubleshooting · TLS cert renewal — Let's Encrypt rate limit"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [devops, tls, certbot, letsencrypt, https]
home_hub: "60-troubleshooting"
related: ["[[devops]]", "[[../network/tls-handshake-fail]]"]
---

# TLS cert renewal 실패 — `too many certificates already issued`

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | devops | 최초 |

## 증상

```
There were too many requests of a given type :: Error creating new order ::
too many certificates already issued for this exact set of domains:
example.com, www.example.com: see https://letsencrypt.org/docs/rate-limits/
```

또는:
```
Failed authorization procedure. Domain not authorized.
```

## 원인

### Let's Encrypt rate limit
- **같은 정확한 도메인 set**: 5 / 7일
- **같은 hostname 의 certificate**: 50 / 7일
- **Pending authorizations**: 300 / 계정
- **Failed validations**: 5 / 시간

→ 같은 cert 를 매번 재발급하는 자동화 스크립트가 hit.

### Domain validation 실패
- HTTP-01: port 80 접근 차단
- DNS-01: DNS provider 인증 부족

## 해결

### 즉시 — Staging endpoint 로 테스트
```bash
certbot --staging certonly -d example.com
# 또는 ACME client 에 staging URL
# https://acme-staging-v02.api.letsencrypt.org/directory
```

→ rate limit 없음. cert 가 staging CA 가 sign — production browser 신뢰 X.

### Wildcard cert (cert 수 절약)
```bash
certbot certonly --dns-route53 -d "*.example.com" -d example.com
```

→ 모든 subdomain 한 cert 로. DNS-01 challenge 필수 (wildcard 는 HTTP-01 불가).

### 1주일 대기
rate limit 은 rolling 7-day window. 1주 후 재시도.

### Auto-renewal 정확히 (재발급 X)
```bash
# 만료 30일 전에만 renewal — already valid 면 skip
certbot renew --dry-run
crontab:
0 0,12 * * * certbot renew --quiet
```

→ `certbot renew` 가 idempotent. `certonly` 매번 호출이 rate limit hit 원인.

### cert-manager (K8s)
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ops@example.com
    privateKeySecretRef: { name: letsencrypt-prod }
    solvers:
      - http01: { ingress: { class: nginx } }
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-tls
spec:
  secretName: example-tls
  issuerRef: { name: letsencrypt-prod, kind: ClusterIssuer }
  dnsNames: [example.com, www.example.com]
```

→ cert-manager 가 renewal 자동 + rate limit 인지.

### AWS ACM (대안)
- Free + auto-renewal + rate limit 없음
- ALB / CloudFront 와 통합

```hcl
resource "aws_acm_certificate" "main" {
  domain_name               = "example.com"
  subject_alternative_names = ["*.example.com"]
  validation_method         = "DNS"
}
```

→ on-prem / VM 환경이면 ACM 불가. Let's Encrypt 또는 ZeroSSL.

## DNS-01 challenge — wildcard / firewalled

```bash
# Route53
certbot certonly --dns-route53 -d "*.example.com"

# Cloudflare
certbot certonly --dns-cloudflare --dns-cloudflare-credentials /etc/cloudflare.ini -d "*.example.com"
```

API credential 가 DNS record 수정 권한 필요.

## 진단

```bash
# Cert 정보
openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | \
  openssl x509 -noout -dates -subject -issuer

# Cert 만료 일자
echo | openssl s_client -connect example.com:443 2>/dev/null | \
  openssl x509 -noout -enddate

# Certbot log
journalctl -u certbot
tail -f /var/log/letsencrypt/letsencrypt.log

# Rate limit 상태 (수동 확인)
# https://crt.sh/?q=example.com
```

## 재발 방지

- `certbot renew` 사용 (만료 30일 전만 renewal). `certonly` 자동화 금지
- cert-manager / ACM 같은 관리 도구
- 모니터링: cert 만료 30일 전 알람
  ```
  - alert: CertExpiringSoon
    expr: probe_ssl_earliest_cert_expiry - time() < 30*86400
  ```

## 관련

- [[devops]] / [[../network/tls-handshake-fail]]
