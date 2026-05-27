---
title: "OrbStack — Docker 사용 (Dockerfile / compose / build)"
kind: knowledge
status: current
created_at: 2026-05-22T12:34:00+09:00
tags: [devops, docker, orbstack, docker-engine]
home_hub: docker
related:
  - "[[getting-started]]"
  - "[[cli]]"
  - "[[../desktop/dashboard]]"
---

# OrbStack — Docker 사용

> OrbStack 의 Docker engine 측 활용. CLI 는 Docker Desktop 과 동일 (`docker` / `docker compose` /
> `docker buildx` 모두). 차이점은 GUI dashboard + 통합 + 성능.

## 1. context 확인

```bash
docker context ls
# orbstack* (자동 선택)
docker info | head -5
# Server: ... OrbStack
```

OrbStack 시작 시 자동으로 docker context 가 `orbstack` 으로 전환.

## 2. Dashboard — Docker 측

좌측 메뉴:

| 메뉴 | 내용 |
| --- | --- |
| **Containers** | 실행 중 / 정지 컨테이너. run/stop/restart/exec/logs/inspect/files browse |
| **Images** | local image. pull/push/run/delete/inspect |
| **Volumes** | named volume. view / export / import / delete |
| **Networks** | bridge / custom / overlay / macvlan |
| **Builds** | 현재 / 과거 build (buildx). step log + cache 통계 |

Docker Desktop 의 Dashboard 와 거의 동일. **차이점**:
- 컨테이너 클릭 시 자동 `orb.local` 도메인 노출 → 브라우저 즉시 열기 가능
- 더 빠른 새로고침 / log streaming

## 3. 기본 명령

```bash
# build
docker build -t myapp:v1 .

# multi-platform (buildx + Rosetta on M1)
docker buildx build --platform linux/amd64,linux/arm64 -t my/app:v1 --push .

# run
docker run -d --name myapp -p 8080:8080 myapp:v1

# OrbStack 만의 boon — orb.local 자동 도메인
curl http://myapp.orb.local            # 컨테이너 직접 접근 (port 80)
curl http://myapp.orb.local:8080       # 노출 port

# compose
docker compose up -d
docker compose watch
docker compose logs -f
```

자세히 → [[networking]] (orb.local 도메인)

## 4. file sync — VirtioFS 성능

bind mount 의 IO 가 Docker Desktop 대비 빠름:

```bash
# Spring Boot dev — host 의 src/ 가 컨테이너 안 hot-reload
docker run -v $PWD:/app -w /app -p 8080:8080 eclipse-temurin:21 ./mvnw spring-boot:run
```

100MB+ 의 file modification 시:
- Docker Desktop (gRPC FUSE): ~2s
- Docker Desktop (VirtioFS): ~0.8s
- **OrbStack (VirtioFS)**: ~0.2s

## 5. Rosetta x86 emulation

Apple Silicon 에서 amd64 image 실행 시 OrbStack 가 자동 Rosetta 활성 (별도 설정 불필요).

```bash
docker run --rm --platform linux/amd64 nginx:alpine uname -m
# x86_64

# 빌드도 빠름
docker buildx build --platform linux/amd64 -t myapp:amd64 .
# Rosetta — Docker Desktop 의 QEMU 보다 5x 빠름 (Desktop 도 Rosetta 옵션 켜면 동등)
```

## 6. buildx — builder 설정

```bash
docker buildx ls
# NAME/NODE      DRIVER/ENDPOINT  STATUS  PLATFORMS
# default*       docker           running linux/arm64, linux/amd64

# 새 builder (multi-platform cache 가속용)
docker buildx create --name multi --use --bootstrap
```

OrbStack 의 default builder 는 이미 multi-platform 지원.

## 7. compose — file sync (watch)

OrbStack 가 Compose v2 watch 기본 지원 — VirtioFS 와 결합 시 매우 빠름:

`compose.yaml`:
```yaml
services:
  api:
    build: .
    develop:
      watch:
        - action: sync
          path: ./src
          target: /app/src
        - action: rebuild
          path: ./pom.xml
```

```bash
docker compose watch
# src/ 변경 → 컨테이너 안 sync (rebuild 없이)
# pom.xml 변경 → 자동 rebuild
```

## 8. 디스크 / 정리

OrbStack disk image 는 **dynamic** — Docker Desktop 의 sparse 64GB 와 달리 사용한 만큼만:

```bash
# 사용량 확인
orb status
# 또는 dashboard 의 좌하단

# 정리
docker system prune -a --volumes        # CLI
# 또는 dashboard > Settings > Reset > "Clean up unused"
```

## 9. private registry

`~/.docker/config.json` 의 auth 가 OrbStack 도 같이 사용:

```bash
# login
docker login ghcr.io -u USERNAME -p $GHCR_TOKEN

# pull
docker pull ghcr.io/org/private:v1
```

회사 사설 registry (HTTPS 인증 없음 — local dev) — `~/.orbstack/config/docker.json` 에 추가:
```json
{
  "insecure-registries": ["registry.local:5000"]
}
```
또는 Settings > Docker > Engine config 에 같은 내용.

## 10. 차이점 한 줄

OrbStack 의 docker engine 은 **Docker 와 100% 호환** + VirtioFS native + 시작 빠름 + dashboard 간소.
docker / compose / buildx CLI 동일. CI / Dockerfile / compose 파일 그대로 사용.

## 11. 다음

- ▶ Linux VM: [[linux-machines]]
- ▶ Kubernetes: [[k8s]]
- ▶ Networking (orb.local): [[networking]]
- ▶ CLI: [[cli]]
- ▶ VS Code Remote 통합: [[integration]]

## 참고

- OrbStack docker docs: https://docs.orbstack.dev/docker/
