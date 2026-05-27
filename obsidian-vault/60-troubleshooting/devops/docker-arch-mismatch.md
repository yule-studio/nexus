---
title: "troubleshooting · docker buildx exec format error (arm64 / amd64)"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [devops, docker, buildx, multi-arch, arm64]
home_hub: "60-troubleshooting"
related: ["[[devops]]"]
---

# `exec format error` — host arch ↔ image arch 불일치

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | devops | 최초 |

## 증상

```
standard_init_linux.go:228: exec user process caused: exec format error
```

또는:
```
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8)
```

- M1/M2/M3 Mac (arm64) 에서 빌드 한 image 가 prod (amd64) 에서 실행 안 됨
- ECS Fargate / EC2 의 arch 와 image arch 불일치

## 원인

container image 는 특정 arch 의 binary 포함. host CPU 와 일치해야 실행 가능. `docker build` default 는 host arch — Mac 에서 빌드하면 arm64 만.

## 해결

### `docker buildx build --platform` 명시
```bash
# 단일 arch
docker buildx build --platform linux/amd64 -t myimage:v1 --load .

# 멀티 arch (registry push 필수)
docker buildx build --platform linux/amd64,linux/arm64 \
  -t myregistry/myimage:v1 --push .
```

→ `--load` 는 단일 arch 만, multi-arch 는 `--push` 만 가능.

### BuildKit driver 설정
```bash
# 새 builder 생성
docker buildx create --use --name multiarch
docker buildx inspect --bootstrap

# qemu 등록 (linux only)
docker run --privileged --rm tonistiigi/binfmt --install all
```

### Dockerfile multi-arch
```dockerfile
# syntax=docker/dockerfile:1.7
FROM --platform=$BUILDPLATFORM node:20 AS build
ARG TARGETPLATFORM
ARG TARGETARCH

WORKDIR /src
COPY . .
RUN echo "Building for $TARGETPLATFORM ($TARGETARCH)"
RUN npm run build

FROM node:20-alpine
COPY --from=build /src/dist /app
CMD ["node", "/app/index.js"]
```

→ build stage 는 host arch (빠름), runtime stage 는 target arch.

### GitHub Actions multi-arch
```yaml
- uses: docker/setup-qemu-action@v3
- uses: docker/setup-buildx-action@v3
- uses: docker/build-push-action@v5
  with:
    platforms: linux/amd64,linux/arm64
    push: true
    tags: myregistry/myimage:latest
```

### ECS — runtimePlatform 명시
```json
{
  "family": "myapp",
  "requiresCompatibilities": ["FARGATE"],
  "runtimePlatform": {
    "cpuArchitecture": "ARM64",            // 또는 X86_64
    "operatingSystemFamily": "LINUX"
  }
}
```

ARM64 Fargate 는 20% 저렴 + 빠름. 단 image 도 arm64 필요.

## Apple Silicon 에서 amd64 image 실행
```bash
docker run --platform linux/amd64 myimage   # rosetta / qemu 사용
```

→ 가능하지만 매우 느림. dev 용만.

## 진단

```bash
# Image arch
docker inspect <image> | jq '.[0].Architecture'
docker manifest inspect <image>

# 실행 host arch
uname -m   # arm64 / x86_64
docker version | grep -i arch
```

## 재발 방지

- 모든 prod image 는 multi-arch build (CI 표준)
- ECS / K8s 의 runtimePlatform 명시
- ARM64 production 시 ARM64 image 만 push (실수 방지)

## 관련

- [[devops]] / [[docker-build-cache-miss]] / [[ecs-task-failed-to-start]]
