---
title: "Docker 실습 05 — BuildKit cache mount + secret"
kind: practice
status: current
created_at: 2026-05-22T12:00:00+09:00
tags: [devops, docker, practice, buildkit, cache, secret]
home_hub: docker
related:
  - "[[practice]]"
  - "[[../dockerfile/buildkit-deep]]"
---

# 실습 05 — BuildKit cache mount + secret

> 시간: 30분. 학습 포인트: `--mount=type=cache` 로 빌드 캐시 영구화 / `--mount=type=secret` 으로 빌드 시 비밀 값 안전 주입 / `--mount=type=bind` 와 `type=ssh`.

## 1. BuildKit 이란?

레거시 docker build 의 **차세대 빌드 엔진**. Docker 23+ 기본. 핵심:

- **DAG 기반** — stage / instruction 의존성 자동 분석 → 병렬 실행
- **content-addressable cache** — 같은 입력 = 같은 cache hit
- **mount 옵션** — `cache` / `secret` / `bind` / `ssh` / `tmpfs`
- **frontend** — Dockerfile 외에 BuildPack / Bazel / 다른 DSL

활성화 확인:
```bash
docker buildx version    # v0.13+
docker buildx ls         # default / desktop-linux builder
```

영상: [BuildKit Deep Dive — Tonis Tiigi](https://www.youtube.com/watch?v=kkpQ_UZn2bo)

## 2. cache mount — 의존성 다운로드 영구화

문제: 매 빌드마다 `mvn dependency:go-offline` / `npm ci` / `apt install` 다시 다운로드.

해결: `--mount=type=cache` 로 빌더 내부 cache 영구화.

### Maven 예

```dockerfile
# syntax=docker/dockerfile:1.7
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app
COPY pom.xml .
RUN --mount=type=cache,target=/root/.m2,sharing=locked \
    mvn -B dependency:go-offline
COPY src ./src
RUN --mount=type=cache,target=/root/.m2,sharing=locked \
    mvn -B package -DskipTests
```

| 옵션 | 의미 |
| --- | --- |
| `target=/root/.m2` | container 내부 mount 경로 |
| `sharing=locked` | 동시 빌드 시 lock (default `shared` — 동시 가능) |
| `id=mvn-cache` | cache 이름 (여러 cache 분리 시) |

### npm 예

```dockerfile
RUN --mount=type=cache,target=/root/.npm \
    npm ci --prefer-offline
```

### apt 예 (debian / ubuntu)

```dockerfile
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends \
      curl ca-certificates && rm -rf /var/lib/apt/lists/*
```

### Go 예

```dockerfile
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -o /out/app ./cmd/api
```

### 검증

```bash
# 첫 빌드 (cache miss)
time docker build --no-cache -t myapp:v1 .
# 약 90초

# 두 번째 (cache hit — pom 안 바뀌었으면)
time docker build -t myapp:v2 .
# 약 5초
```

## 3. secret mount — 비밀 값 안전 주입

문제: API key / SSH key / npm token 을 빌드 시점에만 필요. ENV / ARG 로 넣으면 image layer 에 남음.

해결: `--mount=type=secret` 으로 **빌드 중에만** 마운트, image 에 안 남음.

### 사용

```dockerfile
# syntax=docker/dockerfile:1.7
FROM alpine:3.20
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm install --omit=dev
```

빌드:
```bash
export NPM_TOKEN=...
echo "//registry.npmjs.org/:_authToken=$NPM_TOKEN" > /tmp/npmrc

docker build --secret id=npmrc,src=/tmp/npmrc -t myapp:v1 .

# 또는 env 직접
docker build --secret id=npm,env=NPM_TOKEN -t myapp:v1 .
```

검증 — image 안에 token 없음:
```bash
docker run --rm myapp:v1 cat /root/.npmrc 2>&1
# cat: /root/.npmrc: No such file or directory
```

### SSH agent — 사설 git repo clone

```dockerfile
FROM alpine
RUN apk add --no-cache git openssh
RUN --mount=type=ssh \
    ssh-keyscan github.com >> /etc/ssh/ssh_known_hosts && \
    git clone git@github.com:org/private-repo.git /repo
```

빌드:
```bash
eval $(ssh-agent)
ssh-add ~/.ssh/id_ed25519
docker build --ssh default=$SSH_AUTH_SOCK -t myapp:v1 .
```

## 4. bind mount — 빌드 컨텍스트 일부만

빌드 컨텍스트 전체 복사 대신 일부만:

```dockerfile
RUN --mount=type=bind,source=scripts,target=/tmp/scripts \
    /tmp/scripts/setup.sh
```

장점: `COPY` 보다 빠름 (image layer 에 안 남음). 빌드 시점만 활용 후 폐기.

## 5. tmpfs mount — 빌드 임시 디렉토리

```dockerfile
RUN --mount=type=tmpfs,target=/tmp \
    big-tool --workdir /tmp ...
```

디스크 IO 회피 (메모리).

## 6. cache export / import — CI 가속

```bash
# CI: cache 를 registry 에 push
docker buildx build \
  --cache-to type=registry,ref=ghcr.io/org/myapp:cache,mode=max \
  --cache-from type=registry,ref=ghcr.io/org/myapp:cache \
  -t ghcr.io/org/myapp:v1 \
  --push .
```

| `mode` | 의미 |
| --- | --- |
| `min` (default) | final stage 만 cache |
| `max` | 모든 stage cache (multi-stage 효율 ↑) |

cache type:
- `registry` — registry 에 layer 로 (CI 통합 가장 일반적)
- `gha` — GitHub Actions cache (`type=gha`)
- `local` — 로컬 경로 (`type=local,dest=/tmp/.buildx-cache`)
- `s3` / `azblob` — 클라우드 (large team)

## 7. 검증 — secret leak 없는지

```bash
docker history myapp:v1 --no-trunc | grep -i "secret\|token\|password"
# 출력 없어야 함

docker save myapp:v1 -o /tmp/img.tar
tar -xOf /tmp/img.tar | strings | grep -iE "secret|token|api[_-]?key"
# 빌드 ARG 에 secret 넣으면 여기서 보임 — secret mount 는 안 보임
```

## 8. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| `# syntax=docker/dockerfile:1` 누락 | 1.3+ 필수 (mount 옵션 활성) |
| `DOCKER_BUILDKIT=1` 안 켜짐 (Docker < 23) | 23+ 자동 활성. 이전: `export DOCKER_BUILDKIT=1` |
| ARG 에 secret | 절대 금지 — `--secret` mount 사용 |
| cache mount id 충돌 | 여러 RUN 에 같은 cache 공유 시 `id=mvn-cache` 명시 |
| `sharing=private` 이 어디서 fail | 동시 빌드 시 직렬화. `shared` 또는 `locked` 권장 |

## 9. 다음

- ▶ [[06-multi-platform-build]] — buildx amd64+arm64
- ▶ [[../dockerfile/buildkit-deep]] — BuildKit 내부 (frontend / cache backend / graph)
- ▶ [[../desktop/builder-and-buildkit]] — Desktop GUI

## 참고

- 공식 cache mount: https://docs.docker.com/build/cache/optimize/
- secret mount: https://docs.docker.com/build/building/secrets/
- BuildKit GitHub: https://github.com/moby/buildkit
- 영상: [BuildKit Deep Dive](https://www.youtube.com/watch?v=kkpQ_UZn2bo)
