---
title: "Docker 실습 10 — Image tags / layers / digest / OCI manifest"
kind: practice
status: current
created_at: 2026-05-22T13:15:00+09:00
tags: [devops, docker, practice, image, layer, oci]
home_hub: docker
related:
  - "[[practice]]"
  - "[[../fundamentals/concepts]]"
---

# 실습 10 — Image tags / layers / digest / OCI manifest

> 시간: 30분. 학습 포인트: tag (`v1.2.3`) vs digest (`sha256:abc...`) / immutable referencing /
> layer caching / OCI manifest 구조 / `docker history` / `dive`.

## 1. image 의 정체 — OCI 기준

OCI image 는 4 종 파일:

| 구성 | 내용 |
| --- | --- |
| **manifest** (JSON) | layer + config 의 digest 목록 |
| **config** (JSON) | env / cmd / entrypoint / arch / OS |
| **layer** × N (tar.gz) | filesystem diff (각 RUN/COPY/ADD 별로) |
| **manifest list** (multi-arch) | platform → manifest 매핑 |

```bash
docker pull nginx:1.27-alpine
docker inspect nginx:1.27-alpine | jq '.[0].RootFS.Layers'
# [
#   "sha256:abc...",   # layer 1
#   "sha256:def...",   # layer 2
#   ...
# ]
```

영상: [Docker Image Layers — Bret Fisher](https://www.youtube.com/watch?v=v8RkErCJBPg)

## 2. tag vs digest — immutable referencing

```bash
# tag — 변경 가능 (mutable)
docker pull nginx:1.27-alpine
# 내일 같은 명령 → 다른 image 일 수 있음 (registry 가 tag 갱신)

# digest — 변경 불가 (immutable, content-addressed)
docker pull nginx@sha256:abc123...
# 영원히 같은 image
```

production 에서는 **digest pinning** 권장:

```yaml
# k8s deployment / compose
image: nginx@sha256:abc123...
```

또는 GitOps 로 digest 자동 갱신.

### digest 확인

```bash
docker inspect --format='{{index .RepoDigests 0}}' nginx:1.27-alpine
# nginx@sha256:e2fc938b8...

# image 별 manifest digest
docker buildx imagetools inspect nginx:1.27-alpine
```

## 3. layer caching — Dockerfile 순서가 핵심

각 instruction = 1 layer = 1 hash. 같은 입력 (이전 layer + 현재 instruction) → cache hit.

```dockerfile
# ❌ 나쁜 순서
FROM eclipse-temurin:21-jdk
COPY . /app                # 소스 변경 시 매번 무효화
WORKDIR /app
RUN mvn dependency:go-offline   # ← 매번 재실행 (캐시 miss)
RUN mvn package
```

```dockerfile
# ✅ 좋은 순서
FROM eclipse-temurin:21-jdk
WORKDIR /app
COPY pom.xml .                 # dep 만 먼저
RUN mvn dependency:go-offline  # 캐시 hit (pom 안 변하면)
COPY src ./src                 # 소스 (자주 변경)
RUN mvn package -DskipTests
```

## 4. `docker history` — layer 별 추적

```bash
docker history nginx:1.27-alpine
# IMAGE          CREATED   CREATED BY                                      SIZE
# 4e1b2c3d...    1 day     CMD ["nginx" "-g" "daemon off;"]                0B
# <missing>      1 day     EXPOSE 80                                       0B
# <missing>      1 day     STOPSIGNAL SIGQUIT                              0B
# <missing>      1 day     COPY docker-entrypoint.sh /                     4.62kB
# <missing>      1 day     /bin/sh -c set -x ... apk add nginx ...         7.21MB
# ...

# 자세히 (전체 command)
docker history nginx:1.27-alpine --no-trunc --format "table {{.CreatedBy}}\t{{.Size}}"
```

## 5. dive — layer browser GUI (terminal)

```bash
brew install dive
dive nginx:1.27-alpine
```

각 layer 마다:
- 추가된 파일 (녹색)
- 수정 (노란색)
- 삭제 (빨강 — 실제로는 whiteout marker)
- 효율 (% efficiency)

> 0.95+ 가 좋음. 0.8 미만이면 불필요한 file 많음 (apt cache / 빌드 artifact 등).

## 6. layer 크기 줄이는 패턴

### 6.1 apt cache 정리

```dockerfile
# ❌
RUN apt-get update && apt-get install -y curl
# /var/cache/apt 가 layer 에 남음 (40MB+)

# ✅
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

# 또는 BuildKit cache mount (실습 05)
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y curl
```

### 6.2 단일 RUN 으로 합치기

```dockerfile
# ❌ 3 layer
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# ✅ 1 layer
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*
```

### 6.3 multi-stage (실습 04)

빌드 도구 → builder stage / final 은 runtime 만.

### 6.4 distroless / scratch

```dockerfile
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /out/app /app
ENTRYPOINT ["/app"]
```

shell / libc 없는 최소 image → 15MB.

## 7. manifest list — multi-arch (실습 06)

```bash
docker buildx imagetools inspect nginx:1.27-alpine
# Name:      docker.io/library/nginx:1.27-alpine
# MediaType: application/vnd.oci.image.index.v1+json
# Digest:    sha256:abc...
#
# Manifests:
#   sha256:111... (linux/amd64)
#   sha256:222... (linux/arm64)
#   sha256:333... (linux/arm/v7)
#   sha256:444... (linux/386)
```

`docker pull` 시 클라이언트 arch 자동 매핑. `--platform` 강제 가능.

## 8. registry 의 storage

registry 가 layer 를 **content-addressable** 로 저장:

```
registry/
├── repositories/myapp/
│   ├── _manifests/
│   │   ├── tags/v1.0/
│   │   └── revisions/sha256:abc.../
│   ├── _layers/
│   │   ├── sha256:xxx.../
│   │   ├── sha256:yyy.../
│   │   └── ...
│   └── _uploads/
└── blobs/
    └── sha256/
        ├── ab/abc.../
        └── ...
```

같은 layer 가 여러 image 에서 공유 → 효율적 storage.

## 9. tag 컨벤션

```bash
# semver
myapp:1.2.3
myapp:1.2
myapp:1
myapp:latest                # 변경 가능 — production 비추

# git
myapp:main-abc1234
myapp:v1.2.3-rc1

# date
myapp:2026-05-22

# environment
myapp:prod
myapp:staging
```

production 권장:
- **semver** + **digest pinning**
- `latest` 는 dev 만

## 10. 실습 — 본인 image layer 분석

```bash
# 빌드
docker build -t myapp:v1 .

# layer 별 크기
docker history myapp:v1 --format "table {{.CreatedBy}}\t{{.Size}}" | head

# dive 로 효율 분석
dive myapp:v1
# ↑/↓ Layer 이동, Tab — 파일 트리, Ctrl+F — 검색

# 큰 layer 찾기
docker history myapp:v1 --format "{{.Size}}\t{{.CreatedBy}}" | sort -rh | head -5
```

## 11. 이미지 export / import / save / load

```bash
# image 내보내기 (모든 layer)
docker save myapp:v1 -o myapp-v1.tar
docker load -i myapp-v1.tar

# 컨테이너 → 단일 layer image (rare)
docker export <container> -o snapshot.tar
docker import snapshot.tar myapp:from-container
```

차이:
- `save`/`load`: image (layer 보존)
- `export`/`import`: 컨테이너의 flattened filesystem (1 layer)

## 12. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| `latest` tag 만 사용 | semver + digest pin |
| Dockerfile 의 `RUN apt update` 매번 재실행 | cache mount 또는 단일 RUN |
| layer 가 100+ (instruction 너무 많음) | 합치기 + multi-stage |
| 빌드 시간이 매번 1분+ | `--cache-from` + BuildKit |
| arm64 / amd64 mix 실수 | manifest list (실습 06) |
| `latest` digest 가 매주 변경됨 | mirror 또는 internal registry 로 freeze |

## 13. 다음

- ▶ [[11-registry-docker-hub-and-ghcr]] — push / pull / auth
- ▶ [[12-self-hosted-registry]] — local registry
- ▶ [[../fundamentals/concepts]] — image / layer 이론

## 참고

- OCI image spec: https://github.com/opencontainers/image-spec
- 공식 image: https://docs.docker.com/engine/reference/commandline/image/
- dive: https://github.com/wagoodman/dive
- Bret Fisher Layers: https://www.youtube.com/watch?v=v8RkErCJBPg
