---
title: "Docker 실습 06 — Multi-platform build (amd64 + arm64)"
kind: practice
status: current
created_at: 2026-05-22T12:05:00+09:00
tags: [devops, docker, practice, buildx, multi-platform]
home_hub: docker
related:
  - "[[practice]]"
  - "[[../dockerfile/buildkit-deep]]"
---

# 실습 06 — Multi-platform build (amd64 + arm64)

> 시간: 30분. 학습 포인트: Apple Silicon (arm64) ↔ Linux 서버 (amd64) 한 번에 빌드 / OCI image
> manifest list / QEMU emulation / native cross-build / Docker Desktop 의 Rosetta.

## 1. 왜 multi-platform?

- 개발: Apple Silicon (arm64), 서버: AWS x86 (amd64) → 한 image 가 둘 다
- AWS Graviton (arm64) 비용 절감 시 amd64 + arm64 동시 필요
- 사용자 다양성: 라즈베리 파이 (armv7) / IoT

OCI manifest list 가 같은 tag 안에 여러 platform image 묶음. `docker pull` 시 클라이언트 arch 자동 선택.

영상: [Docker buildx multi-arch — Bret Fisher](https://www.youtube.com/watch?v=hWFLuV1xhFE)

## 2. buildx 활성

```bash
docker buildx version
docker buildx ls
# NAME/NODE       DRIVER/ENDPOINT   STATUS  BUILDKIT  PLATFORMS
# desktop-linux*  docker            running v0.13.0   linux/arm64*, linux/amd64, ...
```

`desktop-linux` builder 가 기본 (Desktop). 다른 builder 생성:

```bash
docker buildx create --name multi --use --bootstrap
docker buildx inspect multi
```

## 3. QEMU emulation 등록 (한 번만)

Apple Silicon 에서 amd64 image 를 빌드하려면 QEMU 필요:

```bash
docker run --privileged --rm tonistiigi/binfmt --install all
# 또는 Desktop 의 'Use Rosetta for x86_64/amd64 emulation' 토글 (Apple Silicon)
```

Linux 호스트:
```bash
sudo apt install qemu-user-static
```

확인:
```bash
docker buildx inspect --bootstrap | grep -i platform
# Platforms: linux/arm64, linux/amd64, linux/amd64/v2, linux/386, linux/arm/v7, linux/arm/v6
```

## 4. 실습 — multi-platform Spring Boot image

`Dockerfile` (이전 실습 04 그대로):

```dockerfile
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn -B dependency:go-offline
COPY src ./src
RUN mvn -B package -DskipTests

FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

빌드 (push 필수 — local 에는 single platform 만 저장 가능):

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/org/myapp:v1 \
  --push .
```

> push 없이 local 에 저장하려면 `--load` — 그러나 multi-platform 은 `--load` 불가. 단일 platform 만:
> ```bash
> docker buildx build --platform linux/arm64 -t myapp:dev --load .
> ```

## 5. 검증 — manifest list

```bash
docker buildx imagetools inspect ghcr.io/org/myapp:v1
# Name:      ghcr.io/org/myapp:v1
# MediaType: application/vnd.docker.distribution.manifest.list.v2+json
# Digest:    sha256:abc...
#
# Manifests:
#   Name:      ghcr.io/org/myapp:v1@sha256:111... (linux/amd64)
#   Name:      ghcr.io/org/myapp:v1@sha256:222... (linux/arm64)
```

각 platform digest 다름 — 같은 tag 가 두 image 가리킴.

pull 시 자동 선택:
```bash
docker pull ghcr.io/org/myapp:v1
docker inspect myapp:v1 | grep Architecture
# "Architecture": "arm64"   (M1 Mac)
# "Architecture": "amd64"   (Intel / 서버)
```

## 6. 빌드 시간 비교

QEMU emulation 은 native 보다 5-10x 느림.

| 빌드 방식 | M1 (arm64) | M1 → amd64 |
| --- | --- | --- |
| native | 1분 | (불가) |
| QEMU | (불필요) | 8분 |
| Rosetta on Apple Silicon | (불필요) | 2분 |

> Apple Silicon: Desktop > Settings > Features in development > "Use Rosetta for x86_64/amd64
> emulation" 켜기 — QEMU 대신 Rosetta → 빠름.

## 7. CI — GitHub Actions multi-arch

`.github/workflows/build.yml`:

```yaml
name: build
on: [push]

jobs:
  multi-arch:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ghcr.io/org/myapp:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

`type=gha` cache 가 매 빌드 가속 (GitHub Actions cache).

## 8. native cross-compile (Go / Rust)

Go 는 cross-compile 가능 — QEMU 없이도 amd64+arm64:

```dockerfile
ARG TARGETOS TARGETARCH
FROM golang:1.23 AS builder
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o /out/app ./cmd/api

FROM gcr.io/distroless/static
COPY --from=builder /out/app /app
ENTRYPOINT ["/app"]
```

`$TARGETOS` / `$TARGETARCH` 는 buildx 가 자동 주입. native compile → 빠름.

## 9. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| `--load` 와 `--platform A,B` 동시 | 불가 — `--push` 사용 |
| QEMU 안 켜져서 amd64 빌드 실패 | `docker run --privileged --rm tonistiigi/binfmt --install all` |
| build 가 매우 느림 | Rosetta on Apple Silicon 활성 + cache 사용 |
| Alpine `apk add` 가 arm64 에서 실패 | base image 가 multi-arch 인지 확인 (`docker buildx imagetools inspect alpine:3.20`) |
| 한쪽 platform 만 push | `--platform linux/amd64` 만 — manifest list 안 됨 |
| native dependency (예: sqlite-jdbc, sharp) | arch 별 binary — package.json 의 optionalDependencies / Maven classifier |

## 10. 검증 / 실행

```bash
# AMD64 에서
docker pull ghcr.io/org/myapp:v1
docker run --rm ghcr.io/org/myapp:v1

# ARM64 에서 같은 tag — 다른 image 자동 pull
docker pull ghcr.io/org/myapp:v1
docker run --rm ghcr.io/org/myapp:v1
```

## 11. 다음

- ▶ [[10-image-tags-and-layers]] — layer / digest / tag policy
- ▶ [[11-registry-docker-hub-and-ghcr]] — GHCR / Docker Hub push
- ▶ [[19-github-actions-build-push]] — CI 통합
- ▶ [[../desktop/builder-and-buildkit]] — Desktop GUI 빌더

## 참고

- 공식: https://docs.docker.com/build/building/multi-platform/
- Bret Fisher 영상: https://www.youtube.com/watch?v=hWFLuV1xhFE
- QEMU + binfmt: https://github.com/tonistiigi/binfmt
