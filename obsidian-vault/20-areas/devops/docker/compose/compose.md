---
title: "Docker Compose — 다중 컨테이너 hub"
kind: knowledge
status: current
created_at: 2026-05-22T13:47:00+09:00
tags: [devops, docker, compose, hub]
home_hub: docker
related:
  - "[[../docker]]"
  - "[[../practice/practice]]"
---

# Docker Compose — 다중 컨테이너 hub

**[[../docker|↑ docker]]**

> 여러 컨테이너 (예: API + DB + Cache + Web) 를 한 YAML 로 정의 + 한 명령 실행. v2 가 표준
> (`docker compose`, no hyphen).

## 1. 학습 순서

| 노트 | 내용 |
| --- | --- |
| [[getting-started]] ★ | compose v2 syntax / service / network / volume / 첫 실행 |

추가 sub doc 후속 작성 예정:
- compose-watch.md (실습 09 가 우선 SSoT)
- compose-profiles.md (실습 09)
- compose-override.md (compose.override.yaml)
- compose-deploy.md (swarm / k8s 변환)

## 2. 핵심 (요약)

`compose.yaml`:
```yaml
services:
  api:
    build: .
    ports: ["8080:8080"]
    environment:
      DB_URL: jdbc:postgresql://db:5432/myapp
    depends_on:
      db: { condition: service_healthy }

  db:
    image: postgres:16
    volumes: [db-data:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]

volumes:
  db-data:
```

```bash
docker compose up -d
docker compose logs -f
docker compose down
```

핵심 명령:
| 명령 | 의미 |
| --- | --- |
| `docker compose up -d` | 백그라운드 시작 |
| `docker compose logs -f` | 모든 service log tail |
| `docker compose exec api bash` | service 진입 |
| `docker compose watch` | dev mode (sync / rebuild) |
| `docker compose down -v` | 정지 + volume 삭제 (주의) |
| `docker compose ps` | 상태 |
| `docker compose config` | rendered yaml (validate) |

## 3. 실습 매핑

| 실습 | doc |
| --- | --- |
| 03 첫 compose | [[../practice/03-compose-full-stack]] |
| 09 watch / profiles / depends_on | [[../practice/09-compose-watch-and-profiles]] |
| 20 종합 프로젝트 | [[../practice/20-end-to-end-bkurs-style]] |

## 4. 학습 영상

| 영상 | 길이 |
| --- | --- |
| [Docker Compose Tutorial — Bret Fisher](https://www.youtube.com/watch?v=2EqarOWMHGw) | 40분 |
| [Compose v2 / Watch — Docker](https://www.youtube.com/watch?v=ApRb38pZyTk) | 15분 |
| [Awesome Compose 예제 모음](https://github.com/docker/awesome-compose) | 라이브러리 |

## 5. 참고

- 공식: https://docs.docker.com/compose/
- compose-spec: https://github.com/compose-spec/compose-spec
- Awesome Compose: https://github.com/docker/awesome-compose
- compose Bridge (k8s 변환): https://docs.docker.com/desktop/features/compose-bridge/

## 6. 관련

- [[../docker]]
- [[../dockerfile/dockerfile]]
- [[../desktop/compose-in-desktop]] (예정)
- [[../practice/practice]]
