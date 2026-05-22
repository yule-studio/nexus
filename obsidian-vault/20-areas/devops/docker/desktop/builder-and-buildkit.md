---
title: "Docker Desktop — Builder / BuildKit / Build Cloud"
kind: knowledge
status: current
created_at: 2026-05-22T14:06:00+09:00
tags: [devops, docker, docker-desktop, buildkit, buildx, build-cloud]
home_hub: docker
related:
  - "[[desktop]]"
  - "[[dashboard]]"
  - "[[../dockerfile/buildkit-deep]]"
---

# Docker Desktop — Builder / BuildKit

> Desktop Dashboard > **Builds** 화면 + buildx + Build Cloud (옵션 유료). multi-platform / cache /
> 진행 시각화.

## 1. Builds Dashboard

좌측 메뉴 **Builds** — 진행 중 / 완료 / 실패 빌드 history.

| 컬럼 | 내용 |
| --- | --- |
| Build name | image tag |
| Status | running / completed / failed |
| Source | context path / Dockerfile |
| Duration | 빌드 시간 |
| Cache | hit rate / cache from |
| Builder | 사용한 builder (default / cloud / 사용자 정의) |

각 빌드 클릭 → 자세한 view:
- step 별 펼치기 (RUN / COPY / FROM)
- 각 step 의 stdout
- cache hit 여부 (각 step 표시)
- 시간 소요

## 2. buildx — Default Builder

Desktop 의 default builder 는 **desktop-linux**. multi-platform 지원 (Apple Silicon 의 Rosetta /
QEMU).

```bash
docker buildx ls
# NAME/NODE       DRIVER/ENDPOINT  STATUS  PLATFORMS
# desktop-linux*  docker           running linux/arm64, linux/amd64, linux/amd64/v2

# 새 builder
docker buildx create --name multi --use --bootstrap
```

## 3. multi-platform build

```bash
# build (push 필수 — local 에는 single platform 만)
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/org/myapp:v1 \
  --push .
```

자세히 → [[../practice/06-multi-platform-build]]

## 4. cache — 4 mode

```bash
# 1. local cache
docker buildx build --cache-to type=local,dest=/tmp/buildx-cache \
  --cache-from type=local,src=/tmp/buildx-cache -t myapp:v1 .

# 2. registry cache
docker buildx build --cache-to type=registry,ref=ghcr.io/org/myapp:cache,mode=max \
  --cache-from type=registry,ref=ghcr.io/org/myapp:cache -t myapp:v1 .

# 3. inline cache (image manifest 안)
docker buildx build --cache-to type=inline -t myapp:v1 --push .

# 4. GitHub Actions cache (CI)
docker buildx build --cache-to type=gha,mode=max --cache-from type=gha -t myapp:v1 .
```

| mode | 의미 |
| --- | --- |
| `min` (default) | final stage 만 cache |
| `max` | 모든 stage cache (multi-stage 효율 ↑) |

## 5. BuildKit mount 옵션

`# syntax=docker/dockerfile:1.7` 명시 후:

```dockerfile
RUN --mount=type=cache,target=/root/.m2 \
    mvn dependency:go-offline
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm install
RUN --mount=type=ssh \
    git clone git@github.com:org/private.git
```

자세히 → [[../practice/05-buildkit-cache-and-secret]]

## 6. Build Cloud (옵션 유료)

Docker Build Cloud — Docker 가 cloud builder 제공:
- multi-platform (amd64 + arm64) **native** (QEMU 대신 진짜 ARM 서버)
- 빠른 cache backend (registry mode 자동)
- CI 통합 (GitHub Actions / GitLab CI)

설정:
```bash
docker buildx create \
  --driver cloud \
  --name my-cloud \
  org/builder-name
docker buildx use my-cloud
docker buildx build --platform linux/amd64,linux/arm64 -t my/app:v1 --push .
```

Docker Hub Pro+ 또는 별도 plan.

## 7. provenance / SBOM 자동

`docker/build-push-action@v6` 의 default 동작:

```yaml
- uses: docker/build-push-action@v6
  with:
    provenance: true       # SLSA provenance attestation 자동
    sbom: true             # SBOM (SPDX) 자동
    push: true
```

attestation 검증:
```bash
docker buildx imagetools inspect my/app:v1
# Manifests + Attestations
```

자세히 → [[../practice/14-sign-and-verify-cosign]]

## 8. builder 관리

```bash
# list
docker buildx ls

# 자세히
docker buildx inspect desktop-linux

# 새로
docker buildx create --name multi --use --bootstrap

# remove
docker buildx rm multi

# 정리 (cache)
docker buildx prune -af
# 또는 GUI: Builds > ⋯ > Clean up builds
```

## 9. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| `docker build` 가 옛 builder (BuildKit X) 사용 | Docker 23+ 자동 BuildKit / `DOCKER_BUILDKIT=1` |
| `--load` 와 `--platform A,B` 동시 사용 | 불가 — `--push` 사용 |
| BuildKit cache mount 안 됨 | `# syntax=docker/dockerfile:1.x` 명시 |
| QEMU 없이 amd64 빌드 fail | `docker run --privileged --rm tonistiigi/binfmt --install all` 또는 Rosetta |
| 디스크 폭주 (build cache 누적) | `docker buildx prune` 또는 daemon.json `builder.gc` |

## 10. daemon.json — build GC

```json
{
  "builder": {
    "gc": {
      "enabled": true,
      "defaultKeepStorage": "20GB"
    }
  }
}
```

→ 빌드 cache 가 20GB 넘으면 자동 정리.

## 11. 영상 / 자료

| 자료 |
| --- |
| [BuildKit Deep Dive — Tonis Tiigi](https://www.youtube.com/watch?v=kkpQ_UZn2bo) |
| [Multi-stage builds — Docker 공식](https://www.youtube.com/watch?v=zpkqNPwEzac) |
| [Docker Build Cloud 소개](https://www.youtube.com/watch?v=4Wia5pVm0a4) |
| [GitHub Actions build-push docs](https://github.com/docker/build-push-action) |

## 12. 다음

- ▶ [[../dockerfile/best-practices]] — Dockerfile 작성
- ▶ [[../dockerfile/buildkit-deep]] — BuildKit graph / cache backend
- ▶ [[../practice/04-multi-stage-build]] — multi-stage
- ▶ [[../practice/05-buildkit-cache-and-secret]] — cache mount + secret
- ▶ [[../practice/06-multi-platform-build]] — multi-arch
- ▶ [[../practice/19-github-actions-build-push]] (예정) — CI

## 13. 참고

- 공식 BuildKit: https://docs.docker.com/build/buildkit/
- buildx: https://docs.docker.com/build/buildx/
- Build Cloud: https://docs.docker.com/build-cloud/
