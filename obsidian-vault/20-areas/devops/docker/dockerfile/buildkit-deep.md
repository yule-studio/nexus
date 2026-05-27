---
title: "BuildKit deep — cache / secret / SBOM / multi-arch"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:15:00+09:00
tags: [devops, docker, buildkit]
---

# BuildKit deep — cache / secret / SBOM / multi-arch

**[[docker|↑ docker]]**

---

## 1. 무엇

```
Docker 의 새 build engine (legacy docker build 대체).

특징:
  - parallel build (layer 들 동시)
  - cache mount (deps cache 영구)
  - secret mount (secret 누설 없음)
  - SSH mount
  - multi-stage 최적
  - multi-arch
  - SBOM / attestation
  - lazy fetch
```

→ `docker build` 의 default 2022+.

---

## 2. enable

```bash
# Docker Desktop / 새 Docker = default
# 옛 Docker:
export DOCKER_BUILDKIT=1
docker build .

# 명시
docker buildx build .

# buildx (별도 도구)
docker buildx version
```

---

## 3. cache mount (★ deps 캐싱)

```dockerfile
# syntax=docker/dockerfile:1.4

FROM gradle:8-jdk21 AS build
WORKDIR /app

# Gradle cache mount (★)
COPY build.gradle.kts settings.gradle.kts ./
RUN --mount=type=cache,target=/root/.gradle \
    gradle dependencies

COPY src/ src/
RUN --mount=type=cache,target=/root/.gradle \
    gradle bootJar

# Maven 도 비슷
# RUN --mount=type=cache,target=/root/.m2 mvn dependency:go-offline

# npm / yarn / pnpm
# RUN --mount=type=cache,target=/root/.npm npm ci

# apt
# RUN --mount=type=cache,target=/var/cache/apt apt-get install -y curl
```

→ cache 가 layer 가 아닌 별도 storage → build 마다 누적.

→ `docker build` 한 번 한 deps 가 두 번째 build 에서 재사용.

---

## 4. secret mount (★)

```dockerfile
# syntax=docker/dockerfile:1.4

FROM alpine:3.18

# GitHub token 으로 private repo
RUN --mount=type=secret,id=github_token \
    GITHUB_TOKEN=$(cat /run/secrets/github_token) && \
    git clone https://${GITHUB_TOKEN}@github.com/myorg/private-repo.git

# AWS credential
RUN --mount=type=secret,id=aws_creds,target=/root/.aws/credentials \
    aws s3 cp s3://my-bucket/file ./
```

```bash
# build 시 secret 전달
docker buildx build \
    --secret id=github_token,src=$HOME/.github/token \
    --secret id=aws_creds,src=$HOME/.aws/credentials \
    .
```

→ image layer 에 secret 안 남음. log 에도 안 나옴.

---

## 5. SSH mount

```dockerfile
# SSH key 통해 private git
RUN --mount=type=ssh \
    git clone git@github.com:myorg/private-repo.git
```

```bash
# build
docker buildx build --ssh default .

# 또는 다른 key
docker buildx build --ssh github_key=$HOME/.ssh/github .
```

---

## 6. multi-arch (★)

```bash
# QEMU 등록 (one-time, ARM build on x86)
docker run --privileged --rm tonistiigi/binfmt --install all

# builder 생성
docker buildx create --use --name multi --driver docker-container

# 한 번에 amd64 + arm64
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    -t myorg/app:1.0 \
    --push \
    .

# 결과: manifest 가 두 image 가리킴
```

→ 같은 image tag → 사용자의 platform 에 맞는 image 자동.

→ k8s 가 ARM64 node + x86 node 둘 다 → 같은 image 사용.

---

## 7. inline cache vs registry cache

```bash
# A. inline cache (image 안에 cache 정보)
docker buildx build \
    --cache-to type=inline \
    --cache-from type=registry,ref=myorg/app:cache \
    -t myorg/app:1.0 \
    --push .

# B. registry cache (★ separate cache image)
docker buildx build \
    --cache-to type=registry,ref=myorg/app:cache,mode=max \
    --cache-from type=registry,ref=myorg/app:cache \
    -t myorg/app:1.0 \
    --push .

# C. local cache
docker buildx build \
    --cache-to type=local,dest=/tmp/cache \
    --cache-from type=local,src=/tmp/cache \
    .

# D. GHA cache
docker buildx build \
    --cache-to type=gha,scope=build,mode=max \
    --cache-from type=gha,scope=build \
    .
```

→ CI 의 build 시간 대폭 단축.

---

## 8. SBOM / provenance (★ 2023+)

```bash
# SBOM 자동 생성
docker buildx build \
    --sbom=true \
    --provenance=true \
    -t myorg/app:1.0 \
    --push .

# attached SBOM 확인
docker buildx imagetools inspect myorg/app:1.0
```

→ image 에 SBOM (SPDX format) + provenance (SLSA) 첨부.

→ supply chain security 의 기본.

---

## 9. bake (★ multi-image build)

```hcl
# docker-bake.hcl
group "default" {
    targets = ["api", "worker", "scheduler"]
}

target "_common" {
    platforms = ["linux/amd64", "linux/arm64"]
    args = {
        BASE_IMAGE = "alpine:3.18"
    }
}

target "api" {
    inherits = ["_common"]
    dockerfile = "Dockerfile.api"
    tags = ["myorg/api:1.0"]
}

target "worker" {
    inherits = ["_common"]
    dockerfile = "Dockerfile.worker"
    tags = ["myorg/worker:1.0"]
}

target "scheduler" {
    inherits = ["_common"]
    dockerfile = "Dockerfile.scheduler"
    tags = ["myorg/scheduler:1.0"]
}
```

```bash
docker buildx bake --push
# → 3 image 병렬 build
```

---

## 10. heredoc syntax (★)

```dockerfile
# syntax=docker/dockerfile:1.4

FROM alpine:3.18

# multi-line script (cleaner)
RUN <<EOF
apk add --no-cache curl python3
curl -L https://example.com/file -o /usr/local/bin/tool
chmod +x /usr/local/bin/tool
EOF

# 또는 다른 shell
RUN <<-EOF bash
    set -e
    if [ ! -d /app ]; then
        mkdir /app
    fi
EOF

# file 작성
COPY <<EOF /etc/config.json
{
    "key": "value"
}
EOF
```

→ 여러 RUN 명령 합치기 (layer ↓) + 가독성 ↑.

---

## 11. RUN --network

```dockerfile
# 일부 RUN 만 network 비활성 (보안)
RUN --network=none echo "no internet"

# 또는 특정 host 만
RUN --network=host curl ...
```

---

## 12. RUN --mount=type=bind

```dockerfile
# 소스 코드 mount (copy 안 함)
RUN --mount=type=bind,source=.,target=/src \
    cp /src/main.py /app/main.py
```

---

## 13. GitHub Actions 통합

```yaml
name: build
on: push

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write
    steps:
      - uses: actions/checkout@v4
      
      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3
      
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ghcr.io/myorg/app:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          sbom: true
          provenance: true
```

---

## 14. 함정

1. **`--mount=type=cache` 의 권한** — root vs non-root user 의 cache.
2. **multi-arch + QEMU** — ARM emulation 느림 (10x). native ARM runner 권장.
3. **inline cache 의 limit** — 큰 cache (>1GB) registry cache 권장.
4. **secret 의 unset** — env var 사용 시 `--mount=type=secret` 으로.
5. **bake 의 file 위치** — workspace root.
6. **multi-stage 의 layer naming** — `AS <name>` 명시.
7. **buildx builder 의 cleanup** — `docker buildx prune` 정기.

---

## 15. 관련

- [[docker|↑ docker]]
- [[dockerfile-best-practices]]
- [[security]]
- [[../security-ops/supply-chain-security|↗ supply chain]]
