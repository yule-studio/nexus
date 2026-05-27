---
title: "Docker 실습 12 — Self-hosted registry (registry:2 + UI + auth + TLS)"
kind: practice
status: current
created_at: 2026-05-22T13:25:00+09:00
tags: [devops, docker, practice, registry, harbor, self-hosted]
home_hub: docker
related:
  - "[[practice]]"
  - "[[../fundamentals/registry]]"
---

# 실습 12 — Self-hosted registry

> 시간: 30분. 학습 포인트: 공식 `registry:2` 띄우기 / 인증 (htpasswd / OIDC) / TLS / UI / Harbor /
> proxy cache (registry mirror) / 운영 패턴.

## 1. 왜 self-hosted?

- 회사 내부 — 외부 인터넷 격리 / 보안 / 컴플라이언스
- Docker Hub rate limit 회피 (proxy mirror)
- 추가 기능 (vulnerability scan / 서명 / replication) — Harbor
- 비용 절감 (대용량 image)

영상: [Self-Hosted Docker Registry — DevOps Toolkit](https://www.youtube.com/watch?v=jhwK8ASYWqg)

## 2. registry:2 — 가장 단순

```bash
# 공식 registry image 띄움
docker run -d --name registry -p 5000:5000 \
  -v registry-data:/var/lib/registry \
  registry:2

# 테스트
docker tag nginx:alpine localhost:5000/nginx:alpine
docker push localhost:5000/nginx:alpine
docker pull localhost:5000/nginx:alpine
```

### insecure registry 등록 (HTTP)

```bash
# Docker Desktop > Settings > Docker Engine
{
  "insecure-registries": ["localhost:5000", "registry.local:5000"]
}
```

restart Desktop. 또는 OrbStack 의 Settings > Docker.

## 3. htpasswd 인증

```bash
# user 생성
docker run --rm --entrypoint htpasswd \
  httpd:2 -Bbn admin secret > auth/htpasswd

# registry with auth
docker run -d --name registry \
  -p 5000:5000 \
  -v $PWD/auth:/auth \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -v registry-data:/var/lib/registry \
  registry:2

# 사용
docker login localhost:5000
# Username: admin
# Password: secret
docker push localhost:5000/nginx:alpine
```

## 4. TLS — Let's Encrypt 또는 자체 서명

### 자체 서명 (개발)

```bash
# CA
openssl req -newkey rsa:4096 -nodes -sha256 -keyout certs/domain.key \
  -addext "subjectAltName = DNS:registry.local" \
  -x509 -days 365 -out certs/domain.crt

# registry with TLS
docker run -d --name registry \
  -p 443:5000 \
  -v $PWD/certs:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:2

# 클라이언트 신뢰
sudo cp certs/domain.crt /etc/docker/certs.d/registry.local/ca.crt
```

### Let's Encrypt (production)

Caddy / Traefik 를 reverse proxy 로:

```yaml
# compose.yaml
services:
  caddy:
    image: caddy:2-alpine
    ports: ["80:80", "443:443"]
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy-data:/data

  registry:
    image: registry:2
    volumes:
      - registry-data:/var/lib/registry
      - ./auth:/auth
    environment:
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd

volumes:
  caddy-data:
  registry-data:
```

`Caddyfile`:
```
registry.example.com {
  reverse_proxy registry:5000
}
```

Caddy 가 Let's Encrypt 자동 — DNS 만 등록.

## 5. proxy / mirror cache

회사 환경 — Docker Hub pull 캐싱:

```bash
docker run -d --name proxy-cache \
  -p 5000:5000 \
  -e REGISTRY_PROXY_REMOTEURL=https://registry-1.docker.io \
  -e REGISTRY_PROXY_USERNAME=$HUB_USER \
  -e REGISTRY_PROXY_PASSWORD=$HUB_PASS \
  -v cache-data:/var/lib/registry \
  registry:2
```

클라이언트 (Docker Desktop / Engine):
```json
{ "registry-mirrors": ["https://registry-cache.corp:5000"] }
```

→ `docker pull nginx` 가 mirror 에서 (cache hit) — Hub rate limit / latency 회피.

## 6. UI — registry GUI

기본 registry:2 는 API only (UI 없음). 추가 UI:

### Joxit registry UI

```yaml
services:
  registry:
    image: registry:2
    # ...

  ui:
    image: joxit/docker-registry-ui:latest
    ports: ["8080:80"]
    environment:
      REGISTRY_TITLE: "My Registry"
      REGISTRY_URL: http://registry:5000
      SINGLE_REGISTRY: "true"
      NGINX_PROXY_PASS_URL: http://registry:5000
```

http://localhost:8080 — 카탈로그 / tag / delete UI.

### 더 풍부한 옵션
- **Harbor** (다음 §)
- **Portus** (deprecated)

## 7. Harbor — production-grade

Harbor 가 가장 풍부한 self-hosted registry:
- web UI + RBAC
- vulnerability scan (Trivy / Clair 통합)
- image signing (Notary v2 / Cosign)
- replication 다중 site
- proxy cache
- audit log

### 설치 (docker-compose)

```bash
wget https://github.com/goharbor/harbor/releases/download/v2.11.0/harbor-offline-installer-v2.11.0.tgz
tar xzf harbor-offline-installer-v2.11.0.tgz
cd harbor
cp harbor.yml.tmpl harbor.yml

# harbor.yml 편집 — hostname / admin password / cert path
sudo ./install.sh --with-trivy --with-chartmuseum
```

`https://harbor.example.com` — 첫 login: `admin / Harbor12345`.

### 사용

```bash
docker login harbor.example.com
docker tag myapp:v1 harbor.example.com/myproject/myapp:v1
docker push harbor.example.com/myproject/myapp:v1
```

Harbor 가 자동 vulnerability scan + 결과 표시. Trivy / Anchore 백엔드.

## 8. distribution backend — S3 / Azure Blob / GCS

대용량 image 의 경우 디스크가 아닌 object storage:

```yaml
# registry config.yml
storage:
  s3:
    accesskey: AKIA...
    secretkey: ...
    region: us-east-1
    bucket: my-registry-bucket
    secure: true
```

```bash
docker run -d -p 5000:5000 \
  -v $PWD/config.yml:/etc/docker/registry/config.yml \
  registry:2
```

## 9. 운영 — garbage collection

```bash
# read-only 모드로 GC 실행
docker exec registry bin/registry garbage-collect \
  --delete-untagged /etc/docker/registry/config.yml

# 또는 dry-run
docker exec registry bin/registry garbage-collect \
  --dry-run /etc/docker/registry/config.yml
```

> registry 정지 / read-only 권장 (GC 중 push 충돌). Harbor 는 자동 schedule.

## 10. 실습 — 종합 (htpasswd + TLS + UI)

`compose.yaml`:

```yaml
services:
  caddy:
    image: caddy:2-alpine
    ports: ["443:443"]
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy-data:/data
    depends_on: [registry, ui]

  registry:
    image: registry:2
    volumes:
      - registry-data:/var/lib/registry
      - ./auth:/auth
    environment:
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: My Registry
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
      REGISTRY_HTTP_HEADERS_Access-Control-Allow-Origin: '["*"]'

  ui:
    image: joxit/docker-registry-ui:latest
    environment:
      REGISTRY_TITLE: Internal Registry
      REGISTRY_URL: https://registry.example.com
      SINGLE_REGISTRY: "true"

volumes:
  registry-data:
  caddy-data:
```

`Caddyfile`:
```
registry.example.com {
  reverse_proxy registry:5000
}
registry-ui.example.com {
  reverse_proxy ui:80
}
```

```bash
htpasswd -Bbn admin secret > auth/htpasswd
docker compose up -d
docker login registry.example.com
docker push registry.example.com/myapp:v1
```

## 11. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| HTTP push 거부 | `insecure-registries` 등록 또는 TLS |
| 디스크 폭주 (image 누적) | garbage collection + lifecycle |
| GC 중 push corruption | read-only 모드 + 짧은 maintenance |
| 인증 안 됨 | htpasswd 가 bcrypt (`-B`) 인지 확인 |
| Harbor 가 메모리 4GB+ 요구 | host RAM 확인 |

## 12. 다음

- ▶ [[13-image-scan-scout-and-trivy]] — CVE scan
- ▶ [[14-sign-and-verify-cosign]] — image signing
- ▶ [[../fundamentals/registry]] — registry 일반

## 참고

- 공식 distribution: https://distribution.github.io/distribution/
- Harbor: https://goharbor.io/
- Joxit UI: https://github.com/Joxit/docker-registry-ui
- DevOps Toolkit: https://www.youtube.com/@DevOpsToolkit
