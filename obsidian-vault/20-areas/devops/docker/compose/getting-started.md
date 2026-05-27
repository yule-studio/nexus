---
title: "docker compose — 다중 컨테이너 관리"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:13:00+09:00
tags: [devops, docker, compose]
---

# docker compose — 다중 컨테이너 관리

**[[docker|↑ docker]]**

---

## 1. 무엇

- 다중 컨테이너 stack 을 YAML 한 파일로 정의 + 실행.
- 로컬 개발 / 단일 노드 production 표준.
- k8s production 전 단계.

---

## 2. 전체 예시 (Spring + PostgreSQL + Redis + nginx)

```yaml
# docker-compose.yml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime
    environment:
      SPRING_PROFILES_ACTIVE: prod
      DB_URL: jdbc:postgresql://postgres:5432/yule
      DB_USER: ${DB_USER:-yule}
      DB_PASSWORD: ${DB_PASSWORD}
      REDIS_HOST: redis
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 3s
      retries: 5
      start_period: 60s
    networks:
      - app-net
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: yule
      POSTGRES_USER: ${DB_USER:-yule}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pg-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-net

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redis-data:/data
    networks:
      - app-net

  nginx:
    image: nginx:1.27-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on:
      - app
    networks:
      - app-net

volumes:
  pg-data:
  redis-data:

networks:
  app-net:
    driver: bridge
```

---

## 3. 자주 쓰는 명령

```bash
docker compose up -d                # start (detached)
docker compose down                  # stop + remove
docker compose down -v               # + volume 삭제
docker compose logs -f app           # 로그 추적
docker compose exec app bash         # 컨테이너 안 진입
docker compose restart app           # 단일 service 재시작
docker compose ps                    # 상태
docker compose pull                  # image 갱신
docker compose build --no-cache app  # rebuild
```

---

## 4. profiles (환경별)

```yaml
services:
  app: ...
  monitoring:
    profiles: ["dev", "monitoring"]
    image: prom/prometheus
  testing:
    profiles: ["test"]
    image: testing-tools
```

```bash
docker compose --profile dev up           # dev profile
docker compose --profile test up          # test profile
```

---

## 5. depends_on + healthcheck

| condition | 의미 |
| --- | --- |
| `service_started` | 컨테이너 시작 (default) |
| `service_healthy` | healthcheck PASS 후 |
| `service_completed_successfully` | exit 0 후 (init 컨테이너용) |

→ DB 가 ready 후 app 시작 = `service_healthy`.

---

## 6. .env 와 secret

```yaml
# 평문 (개발만)
environment:
  DB_PASSWORD: ${DB_PASSWORD}     # .env 에서 읽음

# secret (production)
secrets:
  db_password:
    file: ./secrets/db_password.txt
services:
  app:
    secrets:
      - db_password
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password
```

---

## 7. 함정

1. **`version: '3'`** — 2024+ 는 더 이상 필요 X (Compose v2 자동).
2. **`docker-compose` (붙임표)** — legacy v1 → 새 `docker compose` (띄어쓰기).
3. **depends_on 만 (healthcheck X)** → DB 시작 전 app 시도 → fail.
4. **bind mount + permission** → host UID 와 컨테이너 UID 불일치.
5. **volume cleanup 안 함** → 디스크 폭주 → `docker compose down -v`.

---

## 8. 관련

- [[docker|↑ docker]]
- [[networking]]
- [[volume]]
- [[practice/practice]]
