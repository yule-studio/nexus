---
title: "troubleshooting · Docker build cache miss"
kind: troubleshooting
status: current
created_at: 2026-05-27
tags: [devops, docker, buildx, cache, ci]
home_hub: "60-troubleshooting"
related: ["[[devops]]"]
---

# Docker build cache miss — 매번 전체 layer 재빌드

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-27 | devops | 최초 |

## 증상

- CI 빌드가 매번 10분+ (의존성 install 다시)
- `--cache-from` 명시했는데 효과 없음
- `docker build` log 에 `CACHED` 줄 없음

## 원인

Docker layer cache 는 **이전 layer 가 같을 때만** 다음 layer 캐시. COPY 순서가 자주 바뀌는 파일을 먼저 두면 모든 layer 무효:

```dockerfile
# Bad
FROM node:20
WORKDIR /app
COPY . .                    # ← 매번 모든 source 가 새 hash
RUN npm install             # ← 매번 install (cache miss)
```

## 해결

### Dependency layer 분리
```dockerfile
FROM node:20
WORKDIR /app
COPY package.json package-lock.json ./   # 의존성 파일만 먼저
RUN npm ci                                 # ← package*.json 변경 없으면 cache 적중
COPY . .                                   # source 마지막
RUN npm run build
```

```dockerfile
# Spring Boot
FROM eclipse-temurin:21-jdk AS build
WORKDIR /src
COPY gradlew settings.gradle.kts build.gradle.kts ./
COPY gradle gradle
RUN --mount=type=cache,target=/root/.gradle ./gradlew --no-daemon dependencies || true
COPY . .
RUN --mount=type=cache,target=/root/.gradle ./gradlew --no-daemon bootJar -x test
```

### BuildKit cache mount (gradle / maven / pip / npm)
```dockerfile
# syntax=docker/dockerfile:1.7
FROM node:20
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci
```

→ npm cache 디렉토리가 build 간 유지.

### CI 에 cache-from / cache-to
```yaml
# GitHub Actions
- name: Build
  uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
    # 또는 registry 기반
    # cache-from: type=registry,ref=myregistry/myimage:buildcache
    # cache-to: type=registry,ref=myregistry/myimage:buildcache,mode=max
```

### `.dockerignore` — context 줄임
```
node_modules
.git
*.md
dist
.env*
```

→ COPY . . 의 입력이 작아짐 → cache invalidation 줄어듦.

## Multi-stage build

```dockerfile
FROM node:20 AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:20 AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM nginx:alpine AS runtime
COPY --from=build /app/dist /usr/share/nginx/html
```

각 stage 가 독립적 cache.

## 진단

```bash
DOCKER_BUILDKIT=1 docker build --progress=plain .
# 로그에 [cached] / [hit] 표시

# Cache 상태
docker buildx du
docker buildx prune --filter type=exec.cachemount
```

## 재발 방지

- Dockerfile 표준: deps → source 순서
- BuildKit + cache mount 표준
- CI 에 `cache-from`/`cache-to` 표준
- 빌드 시간 모니터링 — 회귀 즉시 알람

## 관련

- [[devops]] / [[github-actions-cache-miss]]
