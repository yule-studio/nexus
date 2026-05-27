---
title: "Docker Desktop — Compose 통합 / Watch / Bridge"
kind: knowledge
status: current
created_at: 2026-05-22T14:08:00+09:00
tags: [devops, docker, docker-desktop, compose]
home_hub: docker
related:
  - "[[desktop]]"
  - "[[../compose/getting-started]]"
---

# Docker Desktop — Compose 통합

> Desktop 의 GUI 가 compose project 를 stack 으로 보여줌 + Compose v2 watch (dev sync) + Compose
> Bridge (compose → k8s manifest).

## 1. Dashboard 의 compose stack 표시

Containers 화면에서 같은 compose project 의 service 들이 **그룹**으로 묶임:

```
▼ myapp (compose stack)
  ├── myapp-api-1     running     ports 8080:8080
  ├── myapp-db-1      running     pg_isready healthy
  └── myapp-cache-1   running
```

stack 자체에 대한 액션:
- ▶ Start all
- ⏸ Stop all
- 🗑 Delete all (with volumes 옵션)
- View logs (모든 service 통합)

## 2. CLI 통합

CLI 명령은 Desktop 외부에서 그대로:

```bash
docker compose up -d
docker compose logs -f
docker compose down
```

Desktop GUI 가 실시간 동기화 — CLI 변경 즉시 GUI 에 반영.

## 3. Compose v2 — `docker compose` (no hyphen)

| Legacy v1 | v2 (Desktop 내장) |
| --- | --- |
| `docker-compose` (Python) | `docker compose` (Go, Docker CLI 의 subcommand) |
| 별도 binary 설치 | Desktop 자동 |
| 일부 syntax 차이 | 거의 호환 |

v2 권장. 자세히 → [[../compose/compose]]

## 4. compose watch (dev sync)

`compose.yaml`:
```yaml
services:
  api:
    build: .
    develop:
      watch:
        - action: sync                  # file 만 컨테이너로 복사
          path: ./src
          target: /app/src
        - action: rebuild               # 의존성 변경 → image 재빌드
          path: ./pom.xml
```

```bash
docker compose watch
# [api] watch [sync, rebuild]
# - src/Main.java 수정 → 컨테이너 즉시 sync (Spring DevTools / hot-reload)
# - pom.xml 변경 → 자동 rebuild
```

Desktop 의 Builds 화면에 rebuild 진행 표시.

자세히 → [[../practice/09-compose-watch-and-profiles]]

## 5. Compose Bridge (compose → Kubernetes)

Settings > Features in development > **Enable Compose Bridge**:

```bash
# compose.yaml 을 k8s manifest 로 변환
docker compose bridge convert

# 생성:
# ├── api-deployment.yaml
# ├── api-service.yaml
# ├── db-statefulset.yaml
# └── db-pvc.yaml

kubectl apply -f .
```

활용: local dev (compose) → k8s 배포 (제거 없이 prod 같은 환경).

## 6. compose + Dev Environments

`devcontainer.json` 에서 compose 사용:

```jsonc
{
  "name": "bkurs-be dev",
  "dockerComposeFile": "compose.yaml",
  "service": "api",
  "workspaceFolder": "/app",
  "shutdownAction": "stopCompose"
}
```

Dev Environment 가 compose stack 전체 (api / db / cache) 자동 spawn — 자세히 [[dev-environments]]

## 7. profile 으로 환경 분기

```yaml
services:
  api:
    build: .
  db:
    image: postgres:16
  pgadmin:
    image: dpage/pgadmin4
    profiles: [debug]
```

```bash
docker compose up -d                    # default (api + db)
docker compose --profile debug up -d    # + pgadmin
```

Dashboard 는 활성 profile 의 service 만 표시.

## 8. depends_on with health

```yaml
services:
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5
  api:
    depends_on:
      db: { condition: service_healthy }
```

Dashboard 의 stack view 에서 healthy / unhealthy 표시.

자세히 → [[../practice/09-compose-watch-and-profiles]]

## 9. 멀티 compose file (override)

```bash
docker compose -f compose.yaml -f compose.prod.yaml up
# 또는 default override
docker compose up                    # compose.yaml + compose.override.yaml 자동 병합
```

`docker compose config` 로 병합 결과 확인 (Dashboard 에서도 보임).

## 10. CLI cheat

```bash
docker compose ps                      # service 상태
docker compose logs -f api             # 한 service log
docker compose exec api bash           # service 안 진입
docker compose restart api
docker compose pull                    # image pull only
docker compose build                   # build only
docker compose config                  # rendered yaml
docker compose stop                    # stop all (preserve)
docker compose down                    # remove containers + network
docker compose down -v                 # ⚠️ + volume
docker compose down --rmi all          # + image
```

## 11. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| `docker-compose` (v1) syntax 의존 | v2 (`docker compose`) 권장 |
| `depends_on` 가 service_started 만 (default) | `condition: service_healthy` 명시 |
| watch 가 안 동작 | compose 2.22+ + `develop.watch` syntax |
| compose down 으로 prod volume 실수 삭제 | `external: true` named volume |
| Dashboard 의 stack 이 안 묶임 | 모든 service 가 같은 project 이름인지 (`COMPOSE_PROJECT_NAME`) |

## 12. 영상 / 자료

| 자료 |
| --- |
| [Docker Compose Tutorial — Bret Fisher](https://www.youtube.com/watch?v=2EqarOWMHGw) |
| [Compose v2 / Watch — Docker](https://www.youtube.com/watch?v=ApRb38pZyTk) |
| [Compose Bridge to K8s](https://www.docker.com/blog/announcing-docker-compose-bridge-experimental/) |
| [Awesome Compose 예제 모음](https://github.com/docker/awesome-compose) |

## 13. 다음

- ▶ [[../compose/compose]] — compose 일반 hub
- ▶ [[../practice/09-compose-watch-and-profiles]] — 실습
- ▶ [[dev-environments]] — Dev Env + compose
- ▶ [[kubernetes]] — Desktop K8s

## 14. 참고

- 공식 compose: https://docs.docker.com/compose/
- watch: https://docs.docker.com/compose/file-watch/
- Compose Bridge: https://github.com/docker/compose-bridge
