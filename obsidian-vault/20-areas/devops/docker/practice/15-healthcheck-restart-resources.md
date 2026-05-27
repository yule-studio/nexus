---
title: "Docker 실습 15 — HEALTHCHECK / Restart policy / CPU/Memory/PID limits"
kind: practice
status: current
created_at: 2026-05-22T14:25:00+09:00
tags: [devops, docker, practice, healthcheck, restart, resources]
home_hub: docker
related:
  - "[[practice]]"
  - "[[../production/patterns]]"
---

# 실습 15 — HEALTHCHECK / Restart / Resources

> 시간: 30분. 학습 포인트: Dockerfile `HEALTHCHECK` + compose `healthcheck` / restart policy 4종 /
> CPU / memory / pids / read-only filesystem / cgroup v2 / OOM 진단.

## 1. 왜 — production 의 invariant

dev 단계의 "작동" → prod 의 "안정 + 회복":
- 컨테이너가 hang → 자동 감지 (healthcheck) + 재시작 (restart policy)
- 메모리 누수 → host OOM 방어 (memory limit)
- CPU 폭주 → 다른 컨테이너 보호 (cpu limit)
- fork bomb → pid limit

영상: [Production Patterns — Bret Fisher](https://www.youtube.com/watch?v=GwbjRqK01-c)

## 2. HEALTHCHECK

### Dockerfile

```dockerfile
FROM nginx:1.27-alpine
HEALTHCHECK --interval=10s --timeout=3s --start-period=15s --retries=3 \
  CMD wget -q --spider http://localhost/health || exit 1
```

| 옵션 | default | 의미 |
| --- | --- | --- |
| `--interval` | 30s | 검사 주기 |
| `--timeout` | 30s | 검사 자체 timeout |
| `--start-period` | 0s | 시작 후 grace (failure 카운트 안 함) |
| `--retries` | 3 | N회 연속 실패 시 unhealthy |

### compose

```yaml
services:
  api:
    image: myapp
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s
```

### 상태 확인

```bash
docker ps
# CONTAINER ID  IMAGE   STATUS                              PORTS
# abc123        myapp   Up 2 minutes (healthy)              0.0.0.0:8080->8080
# def456        cache   Up 1 minute (health: starting)
# ghi789        worker  Up 30 seconds (unhealthy)

# 자세히
docker inspect --format='{{json .State.Health}}' abc123 | jq
```

### Spring Boot actuator

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure: { include: health,info,prometheus }
  endpoint:
    health:
      probes: { enabled: true }     # /actuator/health/liveness, /readiness
```

```dockerfile
HEALTHCHECK --interval=10s CMD curl -f http://localhost:8080/actuator/health || exit 1
```

## 3. Restart policy

```bash
docker run -d --restart=on-failure:5 myapp           # exit code != 0 시 5회까지
docker run -d --restart=always myapp                 # 항상 (engine reboot 후도)
docker run -d --restart=unless-stopped myapp         # 수동 stop 외 항상
docker run -d --restart=no myapp                     # 안 함 (default)
```

| policy | 의미 |
| --- | --- |
| `no` | 재시작 안 함 |
| `on-failure[:N]` | exit code != 0 시 N회 (default 무한) |
| `always` | engine 재시작 시에도 자동 |
| `unless-stopped` | 명시적 stop 외 always |

compose:
```yaml
services:
  api:
    restart: unless-stopped
```

> production 권장: `unless-stopped` 또는 `on-failure:5`.

## 4. CPU limit

```bash
# core 수
docker run --cpus=2 myapp                # 2 cpu 까지
docker run --cpus=0.5 myapp              # 0.5 cpu (50%)

# cpuset (특정 core)
docker run --cpuset-cpus=0,1 myapp       # core 0,1 만

# weight (relative)
docker run --cpu-shares=512 myapp        # default 1024 의 1/2 우선순위
```

compose:
```yaml
deploy:
  resources:
    limits: { cpus: "2" }
    reservations: { cpus: "0.5" }   # guarantee
```

## 5. Memory limit

```bash
docker run --memory=2g --memory-swap=2g myapp        # swap 비활성 (권장)
docker run --memory=1g --memory-swap=2g myapp        # mem 1g + swap 1g
docker run --memory=1g --memory-reservation=512m     # soft limit
```

| 옵션 | 의미 |
| --- | --- |
| `--memory` (`-m`) | hard limit. 초과 시 OOMKilled |
| `--memory-swap` | mem + swap 합. swap 절대 권장 안 함 — `--memory` 와 같게 |
| `--memory-reservation` | soft limit (kernel 결정) |
| `--oom-kill-disable` | OOM kill 안 함 (위험) |

### JVM 컨테이너 메모리

JVM 이 컨테이너 메모리 인식 (Java 11+):
```bash
docker run -e JAVA_OPTS="-XX:MaxRAMPercentage=75" --memory=2g myapp
# JVM heap 약 1.5GB
```

### OOM 진단

```bash
docker inspect --format='{{.State.OOMKilled}}' <id>
# true

docker events --filter event=oom
# {action: 'oom', ...}

# host 의 kernel log
dmesg | grep -i oom
```

## 6. PID limit (fork bomb 방어)

```bash
docker run --pids-limit=100 myapp
# 컨테이너 안 process 100 개 한도
```

compose:
```yaml
services:
  api:
    deploy:
      resources:
        limits: { pids: 200 }
```

## 7. Read-only filesystem

```bash
docker run --read-only --tmpfs /tmp --tmpfs /run myapp
# / 가 read-only — /tmp /run 만 rw (tmpfs)
```

compose:
```yaml
read_only: true
tmpfs:
  - /tmp:size=100m
  - /run:size=10m
```

> 보안 ↑. 단, app 이 임시 파일 쓰면 tmpfs 필요.

## 8. Capability drop / seccomp

```bash
# 모든 capability 제거 후 필요한 것만
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myapp

# seccomp profile
docker run --security-opt seccomp=/path/to/profile.json myapp

# AppArmor (Ubuntu)
docker run --security-opt apparmor=docker-default myapp

# no privilege escalation
docker run --security-opt no-new-privileges myapp
```

자세히 → [[../fundamentals/security]]

## 9. graceful shutdown

`SIGTERM` 수신 → 진행 요청 완료 → exit. timeout 후 `SIGKILL`.

```bash
docker stop --time=30 myapp                  # 30초 grace (default 10s)
docker run --stop-timeout 30 --stop-signal SIGTERM myapp
```

Spring Boot:
```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 25s
```

> 함정: shell script ENTRYPOINT 가 child process 의 signal 못 받음 → `tini` 사용 (실습 17).

## 10. 실습 — 종합 compose

```yaml
services:
  api:
    image: ghcr.io/seum-running/bkurs-be:v1
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s
    deploy:
      resources:
        limits:   { cpus: "2",  memory: 2G, pids: 200 }
        reservations: { cpus: "0.5", memory: 512M }
    read_only: true
    tmpfs: ["/tmp:size=100m"]
    security_opt:
      - no-new-privileges
    cap_drop: [ALL]
    cap_add: [NET_BIND_SERVICE]
    stop_grace_period: 25s
    environment:
      JAVA_OPTS: "-XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError"

  db:
    image: postgres:16
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
    deploy:
      resources:
        limits: { cpus: "1", memory: 1G }
```

```bash
docker compose up -d
docker compose ps              # health 확인
docker stats                   # 자원 사용

# 부하 테스트
ab -n 10000 -c 100 http://localhost:8080/

# OOM 시뮬
docker exec -it api bash -c "head -c 5G </dev/urandom | tr -d '\0'"
# OOMKilled → 자동 restart
```

## 11. 흔한 함정

| 함정 | 회피 |
| --- | --- |
| HEALTHCHECK 가 너무 짧은 start-period → 시작 중 fail | `start_period: 30s+` |
| restart=always 가 무한 loop | crash log 분석 + on-failure:5 |
| memory limit 안 설정 → host OOM | `--memory` 필수 |
| OOMKilled 인지 모름 | `docker inspect | jq .State.OOMKilled` |
| `--memory-swap` 무제한 | swap 비활성 권장 |
| JVM 이 호스트 메모리 본다 | MaxRAMPercentage 사용 |
| graceful shutdown 작동 안 함 | tini + SIGTERM handler |

## 12. 영상 / 자료

| 자료 |
| --- |
| [Healthcheck + Restart — Bret Fisher](https://www.youtube.com/watch?v=8K_EbWHrmwQ) |
| [Container Resource Limits — Liz Rice](https://www.youtube.com/watch?v=sK5i-N34im8) |
| [JVM memory in containers — Aboullaite](https://www.youtube.com/watch?v=KvBakaTBV-c) |
| [Production Patterns — Bret Fisher](https://www.youtube.com/watch?v=GwbjRqK01-c) |

## 13. 다음

- ▶ [[16-logging-and-monitoring]] — Prometheus / Grafana / Loki
- ▶ [[17-debug-and-troubleshoot]]
- ▶ [[../production/patterns]] — 운영 패턴
- ▶ [[../production/pitfalls]] — PID 1 / signal

## 14. 참고

- 공식 HEALTHCHECK: https://docs.docker.com/engine/reference/builder/#healthcheck
- restart policy: https://docs.docker.com/engine/containers/start-containers-automatically/
- resource limits: https://docs.docker.com/engine/containers/resource_constraints/
