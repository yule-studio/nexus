---
title: "Docker 실습 09 — Compose watch / profiles / depends_on health"
kind: practice
status: current
created_at: 2026-05-22T13:10:00+09:00
tags: [devops, docker, practice, compose, watch]
home_hub: docker
related:
  - "[[practice]]"
  - "[[../compose/getting-started]]"
---

# 실습 09 — Compose watch / profiles / depends_on health

> 시간: 30분. 학습 포인트: `compose watch` (auto sync/rebuild) / profiles (조건부 활성) /
> depends_on with health condition / override file / env_file / extends.

## 1. compose v2 — Docker 23+ 내장

```bash
docker compose version
# Docker Compose version v2.27+
```

`docker-compose` (legacy v1) ≠ `docker compose` (v2). 본 실습은 모두 v2.

영상: [Docker Compose Tutorial — Bret Fisher](https://www.youtube.com/watch?v=2EqarOWMHGw) (40분)

## 2. compose watch — dev mode 자동 sync / rebuild

기존 dev 흐름: 코드 변경 → `docker compose up --build` (느림). watch 는 file 변경 → 자동 sync 또는 rebuild.

`compose.yaml`:

```yaml
services:
  api:
    build: .
    ports: ["8080:8080"]
    develop:
      watch:
        - action: sync                    # file 만 컨테이너로 복사 (rebuild 없음)
          path: ./src
          target: /app/src
        - action: sync+restart            # sync + 컨테이너 restart
          path: ./config
          target: /app/config
        - action: rebuild                 # image 재빌드
          path: ./pom.xml
        - action: rebuild
          path: ./Dockerfile
```

```bash
docker compose watch
# [api] watch [sync, sync+restart, rebuild]
# - src/Main.java 수정 → sync (즉시 반영, hot-reload 가 있으면 자동)
# - pom.xml 변경 → 자동 rebuild
```

> Spring Boot DevTools, Vite HMR, Next.js, Node nodemon 등과 결합 → 빠른 dev loop.

## 3. profiles — 조건부 서비스

특정 환경에서만 일부 service 활성:

```yaml
services:
  api:
    build: .
  db:
    image: postgres:16

  # 옵션 — debug 시만
  pgadmin:
    image: dpage/pgadmin4
    profiles: [debug]
    ports: ["5050:80"]

  # 옵션 — test 시만
  mock-payment:
    image: mockserver/mockserver
    profiles: [test]
```

```bash
docker compose up -d                       # default — api + db
docker compose --profile debug up -d       # api + db + pgadmin
docker compose --profile test up -d        # api + db + mock-payment
docker compose --profile debug --profile test up -d   # 모두
```

활용: dev / test / debug / monitoring stack 분리.

## 4. depends_on with health condition

순서 + 헬스 의존:

```yaml
services:
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  api:
    build: .
    depends_on:
      db:
        condition: service_healthy        # db 가 healthy 일 때만 api 시작

  cache:
    image: redis:7
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s

  worker:
    build: ./worker
    depends_on:
      api:
        condition: service_started        # api 시작 시 (healthy 안 기다림)
      cache:
        condition: service_healthy
```

| condition | 의미 |
| --- | --- |
| `service_started` | depends 가 시작 (default) |
| `service_healthy` | depends 의 HEALTHCHECK 가 통과 |
| `service_completed_successfully` | depends 가 exit 0 (one-shot job) |

## 5. override file — dev / prod 분기

`compose.yaml` (base):
```yaml
services:
  api:
    image: myapp
    ports: ["8080:8080"]
```

`compose.override.yaml` (dev — 자동 적용):
```yaml
services:
  api:
    build: .
    volumes: [".:/app"]
    environment: { DEBUG: "true" }
```

`compose.prod.yaml`:
```yaml
services:
  api:
    image: ghcr.io/org/myapp:v1
    deploy:
      replicas: 3
      resources:
        limits: { cpus: "1.0", memory: 512M }
```

```bash
docker compose up -d                                    # base + override (dev)
docker compose -f compose.yaml -f compose.prod.yaml up  # base + prod
```

## 6. env_file / 환경 변수 우선순위

`.env` (working dir):
```
DB_PASSWORD=secret
TAG=v1.2.3
```

`compose.yaml`:
```yaml
services:
  api:
    image: myapp:${TAG}              # ${TAG} 가 .env 의 v1.2.3
    env_file:
      - ./.env.api                    # service 의 모든 env
    environment:
      DB_PASSWORD: ${DB_PASSWORD}     # .env 의 secret
      ENV: prod                       # plain
```

우선순위 (high → low):
1. `docker compose --env-file ./custom.env up`
2. `environment:` (compose 안)
3. `env_file:` (compose 안)
4. `.env` (working dir)
5. Dockerfile 의 `ENV`

## 7. extends — base service 재사용

`compose.common.yaml`:
```yaml
services:
  base-api:
    build: .
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
    restart: unless-stopped
```

`compose.yaml`:
```yaml
services:
  api-1:
    extends:
      file: compose.common.yaml
      service: base-api
    ports: ["8081:8080"]
  api-2:
    extends:
      file: compose.common.yaml
      service: base-api
    ports: ["8082:8080"]
```

## 8. 실습 — Spring + Postgres + Redis + watch

`compose.yaml`:

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: bkurs_dev
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5

  cache:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s

  api:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/bkurs_dev
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: secret
      SPRING_REDIS_HOST: cache
    ports: ["8080:8080"]
    depends_on:
      db: { condition: service_healthy }
      cache: { condition: service_healthy }
    develop:
      watch:
        - action: sync
          path: ./src
          target: /app/src
        - action: rebuild
          path: ./pom.xml

  pgadmin:
    image: dpage/pgadmin4
    profiles: [debug]
    ports: ["5050:80"]
    environment:
      PGADMIN_DEFAULT_EMAIL: a@a.com
      PGADMIN_DEFAULT_PASSWORD: secret

volumes:
  db-data:
```

실행:
```bash
docker compose watch                       # dev (default profile)
docker compose --profile debug watch       # + pgadmin
docker compose logs -f api                 # 별도 terminal
```

src/ 변경 → 자동 sync → Spring DevTools restart 또는 hot reload.

## 9. config validate / lint

```bash
docker compose config                      # rendered yaml (env / extends 모두 풀린)
docker compose config --quiet              # validate only

# yaml lint
yamllint compose.yaml
```

## 10. 다양한 명령

```bash
# 특정 service 만
docker compose up -d api db
docker compose logs api
docker compose exec api bash
docker compose restart api

# 전체 정리
docker compose down                         # 컨테이너 + network
docker compose down -v                      # + volume
docker compose down --rmi all               # + image
docker compose rm -f                        # stopped service 정리
```

## 11. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| `docker-compose` (v1) 명령 — 옛 syntax 일부 다름 | `docker compose` 권장 (v2) |
| `depends_on` 가 시작 순서만 보장 (healthy 아님) | `condition: service_healthy` 명시 |
| watch 가 안 동작 | `develop.watch` syntax 정확? compose 2.22+ 필요 |
| profile 안 떼면 default 만 | `--profile` 다중 가능 |
| `.env` 변경 후 반영 안 됨 | `down` + `up` 해야 적용 (`restart` 만으로는 안 됨) |

## 12. 다음

- ▶ [[10-image-tags-and-layers]] — image / layer / digest
- ▶ [[11-registry-docker-hub-and-ghcr]] — registry push
- ▶ [[../compose/getting-started]] — compose 일반

## 참고

- 공식: https://docs.docker.com/compose/
- watch: https://docs.docker.com/compose/file-watch/
- Bret Fisher Compose: https://www.youtube.com/watch?v=2EqarOWMHGw
- Awesome Compose: https://github.com/docker/awesome-compose
